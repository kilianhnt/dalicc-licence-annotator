<template>
    <div class="fullscreen" v-bind:class="{ 'dark-bg': readOnly }">
        <v-app-bar dense id="header">
            <v-btn @click="$router.push({name: 'Home'})" icon color="primary" fab>
                <v-icon>mdi-home</v-icon>
            </v-btn>
            <v-toolbar-title>DALICC license annotator</v-toolbar-title>
        </v-app-bar>
        <div id="annotator" class="fullscreen">
            <div id="content">
                <h1>{{readOnly ? 'Retrieve license information' : 'License your work'}}</h1>
                <br/>
                <v-stepper v-model="currentStep" >
                    <v-stepper-header>
                        <v-stepper-step :complete="currentStep > 1" step="1">Define types</v-stepper-step>

                        <v-divider></v-divider>

                        <v-stepper-step v-if="!readOnly" :complete="currentStep > 2" step="2">Define content
                        </v-stepper-step>
                        <v-stepper-step v-if="readOnly" :complete="currentStep > 2" step="2">Check</v-stepper-step>

                        <v-divider v-if="!readOnly"></v-divider>

                        <v-stepper-step v-if="!readOnly" :complete="currentStep > 3" step="3">Start licensing
                        </v-stepper-step>

                    </v-stepper-header>

                    <v-stepper-items>
                        <v-stepper-content step="1">
                            <v-card class="mx-auto" color="lighten-1">
                                <v-container fluid>
                                    <v-toolbar-title>Define types</v-toolbar-title>
                                    <br/>
                                    <v-alert v-if="!readOnly" fluid :value="daliccLicensesApiError" color="red" dark
                                             border="top"
                                             style="text-align: left"
                                             icon="mdi-alert" transition="scale-transition">
                                        Could not recieve data from DALICC license center. Attention, license data could
                                        be outdated!<br/>
                                        <v-btn @click="reloadStore" x-large style="float: right"
                                               :loading="retrying" :disabled="retrying">
                                            <v-icon>mdi-cached</v-icon>
                                            retry
                                        </v-btn>
                                    </v-alert>
                                </v-container>
                                <v-subheader v-if="!readOnly">
                                    Please define what do you want to license and choose an
                                    available license.
                                </v-subheader>
                                <v-subheader v-if="readOnly">
                                    Please define what do you want to check.
                                </v-subheader>
                                <v-container fluid>

                                    <v-select :items="possibleToLicense" id="content-type" filled dense
                                              label="Content type" @change="setContentType"
                                              required prepend-icon="mdi-file"></v-select>

                                    <v-autocomplete v-if="!readOnly" :items="daliccLicenseNames" id="license-type" filled
                                              dense
                                              label="License type" @change="setLicenseType"
                                              required prepend-icon="mdi-copyright"></v-autocomplete>
                                </v-container>

                            </v-card>

                            <!-- TODO: :disabled="!daliccLicensesApiError" -->
                            <v-btn color="primary" @click="evaluate(2)"
                                   :disabled="(!readOnly && (this.contentType === null || this.licenseType === null)) || (readOnly && this.contentType === null)">
                                Continue
                            </v-btn>
                        </v-stepper-content>

                        <v-stepper-content step="2">
                            <v-card class="mx-auto" color="lighten-1">
                                <v-container fluid>
                                    <v-toolbar-title>Define content</v-toolbar-title>
                                    <br/>
                                    <div v-if="this.contentType === this.possibleToLicense[0]">
                                        <v-subheader>Please insert the URL you want to license.
                                        </v-subheader>
                                        <v-text-field id="url" placeholder="https://www.example.com"
                                                      filled v-on:keyup.enter="startCheck" label="URL"
                                                      v-on:keyup="setUrlContent" prepend-icon="mdi-web"></v-text-field>
                                    </div>
                                    <div v-if="this.contentType === this.possibleToLicense[1]">
                                        <v-subheader>Please select the file you want to license.
                                        </v-subheader>
                                        <v-file-input id="file" label="File" filled @change="setFileContent"
                                                      show-size
                                                      prepend-icon="mdi-file"></v-file-input>
                                    </div>
                                    <v-card align="left" :loading="checking" v-if="information.length > 0">
                                        <v-card-subtitle><v-icon>mdi-information</v-icon> Recieved information</v-card-subtitle>
                                        <v-card-text v-html="information"></v-card-text>
                                    </v-card>
                                    <br />
                                    <div v-if="alreadyLicensed && userIsLicensor && userIsLicensor !== null" id="isOwner">Your are the owner!</div>
                                    <div v-if="alreadyLicensed && !userIsLicensor && userIsLicensor !== null" id="isNotOwner">Your are not the owner!</div>
                                </v-container>
                            </v-card>

                            <v-btn color="primary" v-if="!checked || readOnly"
                                   :disabled="(fileContent === null && contentType === possibleToLicense[1]) || (urlContent === null && contentType === possibleToLicense[0])"
                                   :loading="checking" @click="startCheck">Check
                            </v-btn>&nbsp;
                            <v-btn color="primary" v-if="checked && !readOnly && ((userIsLicensor && userIsLicensor !== null && alreadyLicensed) || !alreadyLicensed)"
                                   :disabled="((fileContent === null && contentType === possibleToLicense[1]) || (urlContent === null && contentType === possibleToLicense[0]))"
                                   @click="evaluate(3)">Continue
                            </v-btn>

                            <v-btn text @click="evaluate(1)">Go back</v-btn>
                        </v-stepper-content>

                        <v-stepper-content step="3" v-if="!readOnly">
                            <v-card class="mx-auto" color="lighten-1">
                                <v-container fluid>
                                    <v-toolbar-title>License your content</v-toolbar-title>
                                    <br/>
                                    <v-text-field id="url-summary" v-if="contentType === possibleToLicense[0]"
                                                  filled disabled="disabled" :value="urlContent" label="URL to license"
                                                  prepend-icon="mdi-web"></v-text-field>
                                    <v-text-field id="url-summary" v-if="contentType === possibleToLicense[1]"
                                                  filled disabled="disabled" :value="fileName"
                                                  label="Filename"
                                                  prepend-icon="mdi-file"></v-text-field>

                                    <a target="_blank" :href="licenseTypeUri">
                                        <v-text-field id="url-summary"
                                                      filled disabled="disabled" :value="licenseType"
                                                      label="Selected DALICC license"
                                                      prepend-icon="mdi-copyright"></v-text-field>
                                    </a>
                                    <v-text-field id="url-summary"
                                                  filled disabled="disabled" :value="account"
                                                  label="Your METAMASK account"
                                                  prepend-icon="mdi-wallet"></v-text-field>
                                    <v-textarea id="licensed-output" label="Hash value"
                                                filled disabled="disabled" :value="licenseOutput"
                                                prepend-icon="mdi-information"></v-textarea>
                                </v-container>
                            </v-card>

                            <v-btn color="primary" :loading="licensing" @click="startLicensingProcess">{{licenseOutput.length > 0 || alreadyLicensed ? 'Start override' : 'Start licensing'}}
                            </v-btn>

                            <v-btn text @click="evaluate(2)">Go back</v-btn>
                        </v-stepper-content>


                    </v-stepper-items>
                </v-stepper>
            </div>
        </div>
    </div>
</template>

<script>
    import store from '@/store'
    import EthereumLicense from '@/plugins/SmartContract'

    export default {
        name: 'DaliccLicenseAnnotator',
        data() {
            return {
                readOnly: store.state.readOnly,
                contract: '0xB5D162747E225DCA065e992522c8A9021996c630',
                daliccLicense: null,
                dappBrowser: null,
                publicPath: process.env.BASE_URL,
                currentStep: 0,
                possibleToLicense: ['URL', 'FILE'],
                daliccLicenses: store.state.daliccLicenses,
                daliccLicenseNames: [],
                daliccLicensesApiError: store.state.daliccLicensesApiError,
                retrying: false,
                contentType: null,
                licenseType: null,
                licenseTypeUri: null,
                userIsLicensor: null,
                urlContent: null,
                fileContent: null,
                fileName: null,
                checked: false,
                checking: false,
                information: '',
                licensing: false,
                licensed: false,
                alreadyLicensed: false,
                licenseOutput: '',
                account: null,
                snackbar: false,
            }
        },
        mounted: function () {
            store.commit('fetchDaliccLicenses');
            // Modern dapp/legacy dapp browsers...
            if (window.ethereum || window.web3) {
                this.dappBrowser = true;
            }
            // Non-dapp browsers...
            else {
                this.dappBrowser = false;
            }
            setTimeout(function () {
                document.getElementById('annotator').style.opacity = 1;
            }, 100);
        },
        methods: {
            reloadStore() {
                this.retrying = true;
                store.commit('fetchDaliccLicenses');
                setTimeout(() => this.retrying = false, 500)
            },
            evaluate(step) {
                this.currentStep = step;
                if (this.currentStep < 3) {
                    this.checked = false;
                    this.checking = false;
                    this.userIsLicensor = null;
                    this.information = ''
                }
            },
            startCheck() {
                this.checking = true;
                let data = null;
                if (this.contentType === this.possibleToLicense[0]) {
                    data = this.urlContent;
                } else if (this.contentType === this.possibleToLicense[1]) {
                    data = this.fileContent;
                }
                this.daliccLicense = new EthereumLicense(this.contract);
                this.account = this.daliccLicense.getSelectedAddress();
                this.daliccLicense.getLicenseInformation(data, this.account, (error, result) => {
                    if (!error) {
                        if (!result[0]) {
                            this.information = 'No licensed content detected.';
                            this.alreadyLicensed = false;
                        } else {
                            this.alreadyLicensed = true;
                            const l = this.getLicenseInformationByUrl(result[1]);
                            this.information = 'Licensor: ' + result[2] + '<br />License: <a target="_blank" href="' + l[1] + '">' + l[0] + '</a>';
                            if (typeof this.daliccLicense.getSelectedAddress() === 'string') {
                                this.userIsLicensor = result[2].toLowerCase() === this.daliccLicense.getSelectedAddress().toLowerCase();
                            } else {
                                this.userIsLicensor = false
                            }
                        }
                    } else {
                        this.information = 'Something went wrong. Please retry later.';
                    }
                    this.checking = false;
                    this.checked = true;
                    });
            },
            startLicensingProcess() {
                this.licensing = true;
                let data = null;
                if (this.contentType === this.possibleToLicense[0]) {
                    data = this.urlContent;
                } else if (this.contentType === this.possibleToLicense[1]) {
                    data = this.fileContent;
                }
                this.daliccLicense.license(data, this.licenseTypeUri, this.account, (error, result) => {
                    this.licensing = false;
                    if (!error) {
                        this.licenseOutput = result;
                        this.$router.push({name: 'Home'});
                        this.$store.commit('setSnackbarText', 'Successfully licensed: ' + result);
                        this.$store.commit('setSnackbar', true);
                    } else {
                        this.licenseOutput = 'Something went wrong. Please retry later.';
                    }
                });
            },
            getLicenseInformationByUrl(url) {
                for (let i = 0; i < this.daliccLicenses.length; i++) {
                    if (this.daliccLicenses[i][1] === url) {
                        return this.daliccLicenses[i];
                    }
                }
            },
            setContentType(contentType) {
                this.contentType = contentType;
            },
            setLicenseType(licenseType) {
                this.licenseType = licenseType;
                for (let i = 0; i < this.daliccLicenses.length; i++) {
                    if (this.daliccLicenses[i][0] === this.licenseType) {
                        this.licenseTypeUri = this.daliccLicenses[i][1];
                        break;
                    }
                }
            },
            setUrlContent() {
                this.urlContent = document.getElementById('url').value;
                this.checked = false;
                this.userIsLicensor = null;
                this.information = '';
                if (this.urlContent.length === 0) this.urlContent = null;
            },
            setFileContent() {
                let reader = new FileReader();
                reader.onload = (event) => {
                    this.fileContent = event.target.result;
                    this.checked = false;
                    this.userIsLicensor = null;
                    this.information = '';
                };
                reader.readAsText(document.getElementById('file').files[0]);
                this.fileName = document.getElementById('file').files[0].name;
            }
        },
        created() {
            if (store.state.readOnly === null) {
                this.$route.path.startsWith('/retrieve') ? store.state.readOnly = true : store.state.readOnly = false;
                this.readOnly = store.state.readOnly;
            }
            this.$store.watch(
                (state) => state.daliccLicenses,
                (oldValue, newValue) => {
                    let licenseNames = [];
                    for (let i = 0; i < newValue.length; i++) {
                        licenseNames.push(newValue[i][0]);
                    }
                    this.daliccLicenseNames = licenseNames;
                })
        }
    }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
    h1 {
        text-transform: uppercase;
        text-align: left;
    }

    h1 span {
        font-size: 2em;
    }

    h2 {
        margin-bottom: 2em;
    }

    #annotator button {
        margin-top: 3em;
    }

    h3 {
        margin: 40px 0 0;
    }

    ul {
        list-style-type: none;
        padding: 0;
    }

    li {
        display: inline-block;
        margin: 0 10px;
    }

    a {
        color: #42b983;
    }

    .bottom-center-box {
        position: absolute;
        bottom: 20%;
        left: 10%;
        right: 10%;
        margin: 0 auto;
    }

    .v-stepper__step__step {
        background-color: #42b983 !important;
    }

    #isOwner {
        color: #0f8e0f;
        font-weight: bold;
    }

    #isNotOwner {
        color: #ea4a3c;
        font-weight: bold;
    }
    #header {
        z-index: 100;
    }
    #header i {
        margin: 0px
    }
</style>
