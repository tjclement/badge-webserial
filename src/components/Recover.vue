<template>
    <section id="apps">
        <mdb-row>
            <mdb-col md="12">
                <mdb-card class="mb-4">
                    <mdb-card-header>Update your badge firmware</mdb-card-header>
                    <mdb-card-body>
                        This is an experimental page that you can use to recover the firmware on your Pixel.

                        {{status}}
                        <mdb-btn color="primary" size="sm" v-on:click="recover">Recover your Pixel Badge</mdb-btn>
                    </mdb-card-body>
                </mdb-card>
            </mdb-col>
        </mdb-row>
    </section>
</template>

<script>
    import {
        mdbRow,
        mdbCol,
        mdbBtn,
        mdbTbl,
        mdbCard,
        mdbCardBody,
        mdbCardHeader,
    } from 'mdbvue';

    import {Sketch} from 'vue-color';
    import {
        on_connect,
        readfile,
        createfolder,
        savefile,
        fetch_dir,
        deldir,
        savetextfile,
        writetostdin,
        port,
        writer,
        reader
    } from '../badgecomm';
    import {badges} from '../badge_configs';
    import {EspLoader, inputBuffer} from '../esptool';

    let component = undefined;

    export default {
        name: 'Recover',
        components: {
            mdbRow,
            mdbCol,
            mdbBtn,
            mdbCard,
            mdbCardBody,
            mdbCardHeader,
        },
        beforeMount() {
            component = this;
            let repo_name = '';
            // on_connect().then(async () => {
            //     repo_name = (await transceive('import consts;print(consts.INFO_HARDWARE_WOEZEL_NAME)')).trim();
            //     let version = await transceive('import consts; consts.INFO_FIRMWARE_BUILD');
            //     component.badge_firmware_version = parseInt(version);
            //     component.checking_badge = false;

            //     fetch('https://ota.badge.team/version/'+repo_name+'.txt',{mode:'cors'})
            //         .then(response => {response.json().then((version) => {
            //             component.server_firmware_version = version.build;
            //             component.server_firmware_name = version.name;
            //             component.checking_server = false;
            //         })});
            // });
        },
        methods: {
            render: async (text) => {
                component.status = text;
            },
            getData: async (filename) => {
                var request = new Request('assets/firmware/pixel'+filename);
                var response = await fetch(request);
                return response.arrayBuffer();
            },
            toHex(value, size=4) {
                return "0x" + value.toString(16).toUpperCase().padStart(size, "0");
            },
            showProgress(operation, progress = 0) {
                component.render("Writing "+operation.name+" to " + component.toHex(operation.address) + "... ("+progress+"%)");
            },
            readLoop: async () => {
                while (reader) {
                    const { value, done } = await reader.read();
                    if (done) {
                        reader.releaseLock();
                        break;
                    }
                    addInput(value);
                }
            },
            recover: async (event) => {
                let badge = badges['pixel'];
                component.readLoop().catch((error) => {
                    component.render(this.templates.disconnected, error);
                });
                let espTool = new EspLoader(component.render, (text) => component.render('Error: ' + text), port, badge.flashsize, (data) => writer.write(data));
                let stubLoader;
                component.render("Connecting to the badge...");
                try {
                    if (await espTool.sync()) {
                        if (badge.baudrate != ESP_ROM_BAUD) {
                            component.render("Changing baudrate...");
                            await espTool.setBaudrate(badge.baudrate);
                        }
                        component.render("Connected to " + await espTool.chipName() + " (" + espTool.macAddr().map(value => value.toString(16).toUpperCase().padStart(2, "0")).join(":") + "), loading stub...");
                        stubLoader = await espTool.runStub();
                    } else {
                        component.render("Failed to connect.");
                    }
                } catch(error) {
                    component.render(error);
                    return;
                }

                try {
                    let operations = badge.flash;
                    for (let index = 0; index < operations.length; index++) {
                        let operation = operations[index];
                        component.render("Downloading "+operation.name+"...");
                        let data = await component.getData(operation.filename);
                        component.showProgress(operation);
                        await stubLoader.flashData(data, operation.address, component.showProgress.bind(this, operation));
                    }
                    component.render("Done.");
                } catch(error) {
                    console.error(error);
                    component.render(error);
                }
            }
        },
        data() {
            return {
                status: ''
            }
        },
        computed: {
            filtered_store_apps: () => {
                if(component.selected_store_category === 'all' && component.selected_store_state === 'all') {
                    return component.store_apps;
                }
                return component.store_apps.filter((app) =>
                    (component.selected_store_category === 'all' || app.category === component.selected_store_category) &&
                    (component.selected_store_state === 'all' || app.status === component.selected_store_state));
            }
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    .cascading-admin-card {
        margin: 20px 0;
    }

    .cascading-admin-card .admin-up {
        margin-left: 4%;
        margin-right: 4%;
        margin-top: -20px;
    }

    .cascading-admin-card .admin-up .fas,
    .cascading-admin-card .admin-up .far {
        box-shadow: 0 2px 9px 0 rgba(0, 0, 0, 0.2), 0 2px 13px 0 rgba(0, 0, 0, 0.19);
        padding: 1.7rem;
        font-size: 2rem;
        color: #fff;
        text-align: left;
        margin-right: 1rem;
        border-radius: 3px;
    }

    .cascading-admin-card .admin-up .data {
        float: right;
        margin-top: 2rem;
        text-align: right;
    }

    .admin-up .data p {
        color: #999999;
        font-size: 12px;
    }

    .classic-admin-card .card-body {
        color: #fff;
        margin-bottom: 0;
        padding: 0.9rem;
    }

    .classic-admin-card .card-body p {
        font-size: 13px;
        opacity: 0.7;
        margin-bottom: 0;
    }

    .classic-admin-card .card-body h4 {
        margin-top: 10px;
    }


    .butt {
        content: "";
        border: 0px;
        padding-bottom: 100%;
        display: block;
        width:100%;
        cursor:pointer;
        border-radius: 10px;
        transition: .5s ease;
        backface-visibility: hidden;
        margin: 0 auto;
    }

    .butt:hover {
        opacity: 0.3;
        color:white;
    }

    .button_grid {
        display: grid;
        grid-template-columns: repeat(4, 1fr);
        grid-row-gap: 3.5%;
        grid-column-gap: 3.5%;
    }

    .header {
        margin: 10px;
    }

    .nav-left {
        text-align: left;
        float:left; /* add this */
    }
    .nav-right {
        text-align: right;
    }

    .badge_container {
        width: 80%;
        display: block;
        padding-bottom: 15%;
        margin: 0 auto;
    }

    .page {
        display: flex;
        justify-content: center;
        align-items: center;
    }
</style>
