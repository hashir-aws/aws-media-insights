<template>
  <div>
    <Header :is-upload-active="true" />
    <br />
    <b-container>
      <div v-if="hasAssetParam">
        <a>Running analysis on existing asset: {{ assetIdParam }}</a>
      </div>
      <div v-else>
        <b-alert
          :show="dismissCountDown"
          dismissible
          variant="danger"
          @dismissed="dismissCountDown=0"
          @dismiss-count-down="countDownChanged"
        >
          {{ uploadErrorMessage }}
        </b-alert>
        <b-alert
          :show="showInvalidFile"
          variant="danger"
        >
          {{ invalidFileMessages[invalidFileMessages.length-1] }}
        </b-alert>
        <h1>Upload Content</h1>
        <b-form-file
          v-model="file"
          size="lg"
          placeholder="Choose a file or drop it here..."
          drop-placeholder="Drop file here..."
        >
        </b-form-file>
      </div>
      <br />
      <b-container v-if="isUploading">
        <b-progress :value="uploadValue" show-progress animated></b-progress>
      </b-container>
      <p>{{ description }}</p>
      <b-container>
        <b-row v-if="hasAssetParam">
          <b-col>
            <b-button v-b-toggle.collapse-2 class="m-1">
              Configure Workflow
            </b-button>
            <b-button v-if="validForm" variant="primary" @click="runWorkflow">
              Start Workflow
            </b-button>
            <b-button v-else disabled variant="primary">
              Start Workflow
            </b-button>
          </b-col>
        </b-row>
        <b-row v-else>
          <b-col>
            <b-button v-b-toggle.collapse-2 class="m-1">
              Configure Workflow
            </b-button>
            <b-button
              v-if="validForm"
              variant="primary"
              @click="uploadFiles"
            >
              Start Workflow
            </b-button>
            <b-button v-else disabled variant="primary">
              Start Workflow
            </b-button>
          </b-col>
        </b-row>
      </b-container>
      <br />
      <b-collapse id="collapse-2">
        <b-container class="text-left">
          <b-card-group deck>
            <b-card header="Video and Image Operators">
              <b-form-group>
                <b-form-checkbox-group
                  id="checkbox-group-1"
                  v-model="enabledOperators"
                  :options="videoOperators"
                  name="flavour-1"
                ></b-form-checkbox-group>
                <label>Thumbnail position:</label>
                <b-form-input v-model="thumbnail_position" type="range" min="1" max="20" step="1"></b-form-input>
                {{ thumbnail_position }} sec
                <b-form-input
                  v-if="enabledOperators.includes('faceSearch')"
                  id="Enter face collection id"
                  v-model="faceCollectionId"
                ></b-form-input>

                <b-form-input
                  v-if="enabledOperators.includes('genericDataLookup')"
                  v-model="genericDataFilename"
                  placeholder="Enter data filename"
                ></b-form-input>
              </b-form-group>
              <div v-if="videoFormError" style="color:red">
                {{ videoFormError }}
              </div>
            </b-card>
            <b-card header="Audio Operators">
              <b-form-group>
                <b-form-checkbox-group
                  id="checkbox-group-2"
                  v-model="enabledOperators"
                  :options="audioOperators"
                  name="flavour-2"
                ></b-form-checkbox-group>
                <div v-if="enabledOperators.includes('Transcribe')">
                  <label>Source Language</label>
                  <b-form-select v-model="transcribeLanguage" :options="transcribeLanguages"></b-form-select>
                </div>
              </b-form-group>
              <div v-if="audioFormError" style="color:red">
                {{ audioFormError }}
              </div>
            </b-card>
            <b-card header="Text Operators">
              <b-form-group>
                <b-form-checkbox-group
                  id="checkbox-group-3"
                  v-model="enabledOperators"
                  :options="textOperators"
                  name="flavour-3"
                ></b-form-checkbox-group>
                <div v-if="enabledOperators.includes('Translate')">
                  <label>Translation Source Language</label>
                  <b-form-select v-model="transcribeLanguage" :options="transcribeLanguages"></b-form-select>
                  <label>Translation Target Language</label>
                  <b-form-select v-model="targetLanguageCode" :options="translateLanguages"></b-form-select>
                </div>
              </b-form-group>
              <div v-if="textFormError" style="color:red">
                {{ textFormError }}
              </div>
            </b-card>
          </b-card-group>
          <div align="right">
            <button type="button" class="btn btn-link" @click="selectAll">
              Select All
            </button>
            <button type="button" class="btn btn-link" @click="clearAll">
              Clear All
            </button>
          </div>
        </b-container>
      </b-collapse>
    </b-container>
    <b-container v-if="executed_assets.length > 0">
      <label>Execution History</label>
      <b-table
        :fields="fields"
        bordered
        hover
        small
        responsive
        show-empty
        fixed
        :items="executed_assets"
      >
        <template v-slot:cell(workflow_status)="data">
          <a v-if="data.item.workflow_status !== 'Queued'"
            href
            @click.stop.prevent="openWindow(data.item.state_machine_console_link)"
          >{{ data.item.workflow_status }}</a>
          <div v-if="data.item.workflow_status === 'Queued'">{{ data.item.workflow_status }}</div>
        </template>
      </b-table>
      <b-button size="sm" @click="clearHistory">
        Clear History
      </b-button>
    </b-container>
  </div>
</template>

<script>
import Header from "@/components/Header.vue";
import { mapState } from "vuex";

export default {
  components: {
    Header
  },
  data() {
    return {
      file: null,
      isUploading: null,
      uploadValue: null,
      valid_media_types: [
        "cmaf",
        "dash",
        "hls",
        "mp4",
        "f4v",
        "mxf",
        "mov",
        "ismv",
        "raw",
        "av1",
        "avc",
        "hevc",
        "mpeg-2",
        "avi",
        "mkv",
        "webm"
      ], // see https://docs.aws.amazon.com/mediaconvert/latest/ug/reference-codecs-containers.html
      fields: [
        {
          asset_id: {
            label: "Asset Id",
            sortable: false
          }
        },
        {
          file_name: {
            label: "File Name",
            sortable: false
          }
        },
        {
          workflow_status: {
            label: "Workflow Status",
            sortable: false
          }
        }
      ],
      thumbnail_position: 10,
      hasAssetParam: false,
      assetIdParam: "",
      enabledOperators: [
        "labelDetection",
        "celebrityRecognition",
        "textDetection",
        "contentModeration",
        "faceDetection",
        "thumbnail",
        "Transcribe",
        "Translate",
        "ComprehendKeyPhrases",
        "ComprehendEntities",
        "shotDetection",
        "technicalCueDetection"
      ],
      videoOperators: [
        { text: "Object Detection", value: "labelDetection" },
        { text: "Technical Cue Detection", value: "technicalCueDetection" },
        { text: "Shot Detection", value: "shotDetection" },
        { text: "Celebrity Recognition", value: "celebrityRecognition" },
        { text: "Content Moderation", value: "contentModeration" },
        { text: "Face Detection", value: "faceDetection" },
        { text: "Word Detection", value: "textDetection" },
        { text: "Face Search", value: "faceSearch" },
        { text: "Generic Data Lookup (video only)", value: "genericDataLookup" }
      ],
      audioOperators: [{ text: "Transcribe", value: "TranscribeVideo" }],
      textOperators: [
        { text: "Comprehend Key Phrases", value: "ComprehendKeyPhrases" },
        { text: "Comprehend Entities", value: "ComprehendEntities" },
        { text: "Translate", value: "Translate" }
      ],
      faceCollectionId: "",
      genericDataFilename: "",
      transcribeLanguage: "en-US",
      transcribeLanguages: [
        { text: "Arabic, Gulf", value: "ar-AE" },
        { text: "Arabic, Modern Standard", value: "ar-SA" },
        { text: "Chinese Mandarin", value: "zh-CN" },
        { text: "Dutch", value: "nl-NL" },
        { text: "English, Australian", value: "en-AU" },
        { text: "English, British", value: "en-GB" },
        { text: "English, Indian-accented", value: "en-IN" },
        { text: "English, Irish", value: "en-IE" },
        { text: "English, Scottish", value: "en-AB" },
        { text: "English, US", value: "en-US" },
        { text: "English, Welsh", value: "en-WL" },
        // Disabled until 'fa' supported by AWS Translate
        // {text: 'Farsi', value: 'fa-IR'},
        { text: "French", value: "fr-FR" },
        { text: "French, Canadian", value: "fr-CA" },
        { text: "German", value: "de-DE" },
        { text: "German, Swiss", value: "de-CH" },
        { text: "Hebrew", value: "he-IL" },
        { text: "Hindi", value: "hi-IN" },
        { text: "Indonesian", value: "id-ID" },
        { text: "Italian", value: "it-IT" },
        { text: "Japanese", value: "ja-JP" },
        { text: "Korean", value: "ko-KR" },
        { text: "Malay", value: "ms-MY" },
        { text: "Portuguese", value: "pt-PT" },
        { text: "Portuguese, Brazilian", value: "pt-BR" },
        { text: "Russian", value: "ru-RU" },
        { text: "Spanish", value: "es-ES" },
        { text: "Spanish, US", value: "es-US" },
        { text: "Tamil", value: "ta-IN" },
        // Disabled until 'te' supported by AWS Translate
        // {text: 'Telugu', value: 'te-IN'},
        { text: "Turkish", value: "tr-TR" }
      ],
      translateLanguages: [
        { text: "Afrikaans", value: "af" },
        { text: "Albanian", value: "sq" },
        { text: "Amharic", value: "am" },
        { text: "Arabic", value: "ar" },
        { text: "Azerbaijani", value: "az" },
        { text: "Bengali", value: "bn" },
        { text: "Bosnian", value: "bs" },
        { text: "Bulgarian", value: "bg" },
        { text: "Chinese (Simplified)", value: "zh" },
        // AWS Translate does not support translating from zh to zh-TW
        // {text: 'Chinese (Traditional)', value: 'zh-TW'},
        { text: "Croatian", value: "hr" },
        { text: "Czech", value: "cs" },
        { text: "Danish", value: "da" },
        { text: "Dari", value: "fa-AF" },
        { text: "Dutch", value: "nl" },
        { text: "English", value: "en" },
        { text: "Estonian", value: "et" },
        { text: "Finnish", value: "fi" },
        { text: "French", value: "fr" },
        { text: "French (Canadian)", value: "fr-CA" },
        { text: "Georgian", value: "ka" },
        { text: "German", value: "de" },
        { text: "Greek", value: "el" },
        { text: "Hausa", value: "ha" },
        { text: "Hebrew", value: "he" },
        { text: "Hindi", value: "hi" },
        { text: "Hungarian", value: "hu" },
        { text: "Indonesian", value: "id" },
        { text: "Italian", value: "it" },
        { text: "Japanese", value: "ja" },
        { text: "Korean", value: "ko" },
        { text: "Latvian", value: "lv" },
        { text: "Malay", value: "ms" },
        { text: "Norwegian", value: "no" },
        { text: "Persian", value: "fa" },
        { text: "Pashto", value: "ps" },
        { text: "Polish", value: "pl" },
        { text: "Portuguese", value: "pt" },
        { text: "Romanian", value: "ro" },
        { text: "Russian", value: "ru" },
        { text: "Serbian", value: "sr" },
        { text: "Slovak", value: "sk" },
        { text: "Slovenian", value: "sl" },
        { text: "Somali", value: "so" },
        { text: "Spanish", value: "es" },
        { text: "Swahili", value: "sw" },
        { text: "Swedish", value: "sv" },
        { text: "Tagalog", value: "tl" },
        { text: "Tamil", value: "ta" },
        { text: "Thai", value: "th" },
        { text: "Turkish", value: "tr" },
        { text: "Ukrainian", value: "uk" },
        { text: "Urdu", value: "ur" },
        { text: "Vietnamese", value: "vi" }
      ],
      sourceLanguageCode: "en",
      targetLanguageCode: "es",
      uploadErrorMessage: "",
      invalidFileMessage: "",
      invalidFileMessages: [],
      showInvalidFile: false,
      dismissSecs: 8,
      dismissCountDown: 0,
      executed_assets: [],
      workflow_status_polling: null,
      description:
        "Click Start Workflow to begin. Content analysis status will be shown after the workflow is started.",
      s3_destination: "s3://" + this.DATAPLANE_BUCKET,
    };
  },
  computed: {
    ...mapState(["execution_history"]),
    textFormError() {
      return "";
    },
    audioFormError() {
      // Validate transcribe is enabled if any text operator is enabled
      if (
        !this.enabledOperators.includes("Transcribe") &&
        (this.enabledOperators.includes("Translate") ||
          this.enabledOperators.includes("ComprehendEntities") ||
          this.enabledOperators.includes("ComprehendKeyPhrases"))
      ) {
        return "Transcribe must be enabled if any text operator is enabled.";
      }
      return "";
    },
    videoFormError() {
      // Validate face collection ID if face search is enabled
      if (this.enabledOperators.includes("faceSearch")) {
        // Validate that the collection ID is defined
        if (this.faceCollectionId === "") {
          return "Face collection name is required.";
        }

        // Validate that the collection ID matches required regex
        else if (new RegExp("[^a-zA-Z0-9_.\\-]").test(this.faceCollectionId)) {
          return "Face collection name must match pattern [a-zA-Z0-9_.\\\\-]+";
        }
        // Validate that the collection ID is not too long
        else if (this.faceCollectionId.length > 255) {
          return "Face collection name must have fewer than 255 characters.";
        }
      }
      if (this.enabledOperators.includes("genericDataLookup")) {
        // Validate that the collection ID is defined
        if (this.genericDataFilename === "") {
          return "Data filename is required.";
        }
        // Validate that the collection ID matches required regex
        else if (!new RegExp("^.+\\.json$").test(this.genericDataFilename)) {
          return "Data filename must have .json extension.";
        }
        // Validate that the data filename is not too long
        else if (this.genericDataFilename.length > 255) {
          return "Data filename must have fewer than 255 characters.";
        }
      }
      return "";
    },
    validForm() {
      let validStatus = true;
      if (
        this.invalid_file_types ||
        this.textFormError ||
        this.audioFormError ||
        this.videoFormError
      )
        validStatus = false;
      return validStatus;
    },
    workflowConfig() {
      return {
        Name: "CasVideoWorkflow",
        Configuration: {
          defaultPrelimVideoStage: {
            Thumbnail: {
              ThumbnailPosition: this.thumbnail_position.toString(),
              Enabled: true
            },
            Mediainfo: {
              Enabled: true
            }
          },
          defaultVideoStage: {
            faceDetection: {
              Enabled: this.enabledOperators.includes("faceDetection")
            },
            technicalCueDetection: {
              Enabled: this.enabledOperators.includes("technicalCueDetection")
            },
            shotDetection: {
              Enabled: this.enabledOperators.includes("shotDetection")
            },
            celebrityRecognition: {
              Enabled: this.enabledOperators.includes("celebrityRecognition")
            },
            labelDetection: {
              Enabled: this.enabledOperators.includes("labelDetection")
            },
            Mediaconvert: {
              Enabled: false
            },
            contentModeration: {
              Enabled: this.enabledOperators.includes("contentModeration")
            },
            faceSearch: {
              Enabled: this.enabledOperators.includes("faceSearch"),
              CollectionId:
                this.faceCollectionId === ""
                  ? "undefined"
                  : this.faceCollectionId
            },
            textDetection: {
              Enabled: this.enabledOperators.includes("textDetection")
            },
            GenericDataLookup: {
              Enabled: this.enabledOperators.includes("genericDataLookup"),
              Bucket: this.DATAPLANE_BUCKET,
              Key:
                this.genericDataFilename === ""
                  ? "undefined"
                  : this.genericDataFilename
            }
          },
          defaultAudioStage: {
            TranscribeVideo: {
              Enabled: this.enabledOperators.includes("Transcribe"),
              TranscribeLanguage: this.transcribeLanguage
            }
          },
          defaultTextStage: {
            Translate: {
              Enabled: this.enabledOperators.includes("Translate"),
              SourceLanguageCode: this.transcribeLanguage.split("-")[0],
              TargetLanguageCode: this.targetLanguageCode
            },
            ComprehendEntities: {
              Enabled: this.enabledOperators.includes("ComprehendEntities")
            },
            ComprehendKeyPhrases: {
              Enabled: this.enabledOperators.includes("ComprehendKeyPhrases")
            }
          },
          defaultTextSynthesisStage: {
            // Polly is available in the MIECompleteWorkflow but not used in the front-end, so we've disabled it here.
            Polly: {
              Enabled: false
            }
          }
        }
      };
    }
  },
  created: function() {
    if (this.$route.query.asset) {
      this.hasAssetParam = true;
      this.assetIdParam = this.$route.query.asset;
    }
  },
  mounted: function() {
    this.executed_assets = this.execution_history;
    this.pollWorkflowStatus();
    //console.log("this.DATAPLANE_BUCKET: " + this.DATAPLANE_BUCKET)
  },
  beforeDestroy() {
    clearInterval(this.workflow_status_polling);
  },
  methods: {
    selectAll: function() {
      this.enabledOperators = [
        "labelDetection",
        "textDetection",
        "celebrityRecognition",
        "contentModeration",
        "faceDetection",
        "thumbnail",
        "Transcribe",
        "Translate",
        "ComprehendKeyPhrases",
        "ComprehendEntities",
        "technicalCueDetection",
        "shotDetection"
      ];
    },
    clearAll: function() {
      this.enabledOperators = [];
    },
    openWindow: function(url) {
      window.open(url);
    },
    countDownChanged(dismissCountDown) {
      this.dismissCountDown = dismissCountDown;
    },
    s3UploadError(error) {
      console.log(error);
      // display alert
      this.uploadErrorMessage = error;
      this.dismissCountDown = this.dismissSecs;
    },

    // TODO: Update these methods to use events from vue bootstrap file picker component

    // fileAdded: function(file) {
    //   let errorMessage = "";
    //   if (
    //     !file.type.match(/image\/.+|video\/.+|application\/json/g) &&
    //     !this.valid_media_types.includes(
    //       file.name
    //         .split(".")
    //         .pop()
    //         .toLowerCase()
    //     )
    //   ) {
    //     if (file.type === "") {
    //       console.log("here");
    //       errorMessage = "Unsupported file type: unknown";
    //     } else errorMessage = "Unsupported file type: " + file.type;
    //     this.invalidFileMessages.push(errorMessage);
    //     this.showInvalidFile = true;
    //   }
    // },
    // fileRemoved: function(file) {
    //   let errorMessage = "";
    //   if (!file.type.match(/image\/.+|video\/.+|application\/json/g)) {
    //     if (file.type === "") errorMessage = "Unsupported file type: unknown";
    //     else errorMessage = "Unsupported file type: " + file.type;
    //   }
    //   this.invalidFileMessages = this.invalidFileMessages.filter(function(
    //     value
    //   ) {
    //     return value != errorMessage;
    //   });
    //   if (this.invalidFileMessages.length === 0) this.showInvalidFile = false;
    // },

    runWorkflow: async function(location) {
      const vm = this;
      let media_type = null;
      let s3Key = null;
      if ("key" in location) {
        media_type = location.type;
        s3Key = 'public/' + location.key; // add in public since amplify prepends that to all keys
      } else {
        media_type = this.$route.query.mediaType;
        s3Key = this.$route.query.s3key.split("/").pop();
      }
      let data = {};
      if (this.hasAssetParam) {
        if (media_type === "video") {
          data = vm.workflowConfig;
          data["Input"] = { AssetId: this.assetIdParam, Media: { Video: {} } };
        } else if (media_type === "image") {
          data = {
            Input: {
              AssetId: this.assetIdParam,
              Media: {
                Image: {}
              }
            },
            Name: "CasImageWorkflow",
            Configuration: {
              ValidationStage: {
                MediainfoImage: {
                  Enabled: true
                }
              },
              RekognitionStage: {
                faceSearchImage: {
                  Enabled: this.enabledOperators.includes("faceSearch"),
                  CollectionId:
                    this.faceCollectionId === ""
                      ? "undefined"
                      : this.faceCollectionId
                },
                labelDetectionImage: {
                  Enabled: this.enabledOperators.includes("labelDetection")
                },
                textDetectionImage: {
                  Enabled: this.enabledOperators.includes("textDetection")
                },
                celebrityRecognitionImage: {
                  Enabled: this.enabledOperators.includes(
                    "celebrityRecognition"
                  )
                },
                contentModerationImage: {
                  Enabled: this.enabledOperators.includes("contentModeration")
                },
                faceDetectionImage: {
                  Enabled: this.enabledOperators.includes("faceDetection")
                }
              }
            }
          };
        } else {
          vm.s3UploadError(
            "Unsupported media type, " + this.$route.query.mediaType + "."
          );
        }
      } else {
        if (media_type.match(/image/g)) {
          data = {
            Name: "CasImageWorkflow",
            Configuration: {
              ValidationStage: {
                MediainfoImage: {
                  Enabled: true
                }
              },
              RekognitionStage: {
                faceSearchImage: {
                  Enabled: this.enabledOperators.includes("faceSearch"),
                  CollectionId:
                    this.faceCollectionId === ""
                      ? "undefined"
                      : this.faceCollectionId
                },
                labelDetectionImage: {
                  Enabled: this.enabledOperators.includes("labelDetection")
                },
                textDetectionImage: {
                  Enabled: this.enabledOperators.includes("textDetection")
                },
                celebrityRecognitionImage: {
                  Enabled: this.enabledOperators.includes(
                    "celebrityRecognition"
                  )
                },
                contentModerationImage: {
                  Enabled: this.enabledOperators.includes("contentModeration")
                },
                faceDetectionImage: {
                  Enabled: this.enabledOperators.includes("faceDetection")
                }
              }
            },
            Input: {
              Media: {
                Image: {
                  S3Bucket: this.DATAPLANE_BUCKET,
                  S3Key: s3Key
                }
              }
            }
          };
        } else if (
          media_type.match(/video/g) ||
          this.valid_media_types.includes(
            s3key
              .split(".")
              .pop()
              .toLowerCase()
          )
        ) {
          data = vm.workflowConfig;
          data["Input"] = {
            Media: {
              Video: {
                S3Bucket: this.DATAPLANE_BUCKET,
                S3Key: s3Key
              }
            }
          };
        } else if (media_type === "application/json") {
          // JSON files may be uploaded for the genericDataLookup operator, but
          // we won't run a workflow for json file types.
          //console.log("Data file has been uploaded to s3://" + location.s3ObjectLocation.fields.key);
          return;
        } else {
          vm.s3UploadError("Unsupported media type: " + media_type + ".");
        }
      }
      //console.log(JSON.stringify(data));

      // TODO: Should this be its own function?

      let apiName = 'mieWorkflowApi'
      let path = 'workflow/execution'
      let requestOpts = {
          headers: {
            'Content-Type': 'application/json'
          },
          response: true,
          body: data,
          queryStringParameters: {} // optional
      };
      try {
        let response = await this.$Amplify.API.post(apiName, path, requestOpts);
        let asset_id = response.data.AssetId;
        let wf_id = response.data.Id;
        //console.log("Media assigned asset id: " + asset_id);
        let executed_asset = {
            asset_id: asset_id,
            file_name: s3Key,
            workflow_status: "",
            state_machine_console_link: "",
            wf_id: wf_id
          };
        vm.executed_assets.push(executed_asset);
        vm.getWorkflowStatus(wf_id);
        this.hasAssetParam = false;
        this.assetIdParam = "";
      } catch (error) {
        alert(
          "ERROR: Failed to start workflow. Check Workflow API logs."
        );
        console.log(error)
      }
    },
    async getWorkflowStatus(wf_id) {
      const vm = this;
      let apiName = 'mieWorkflowApi'
      let path =  "workflow/execution/" + wf_id
      let requestOpts = {
        headers: {},
        response: true,
        queryStringParameters: {} // optional
      };
      try {
        let response = await this.$Amplify.API.get(apiName, path, requestOpts);
        for (let i = 0; i < vm.executed_assets.length; i++) {
          if (vm.executed_assets[i].wf_id === wf_id) {
            vm.executed_assets[i].workflow_status = response.data.Status;
            vm.executed_assets[i].state_machine_console_link =
            "https://" + this.AWS_REGION + ".console.aws.amazon.com/states/home?region=" + this.AWS_REGION + "#/executions/details/" + response.data.StateMachineExecutionArn;
            break;
          }
        }
      this.$store.commit("updateExecutedAssets", vm.executed_assets);
      } catch (error) {
        alert("ERROR: Failed to get workflow status");
        console.log(error)
      }
    },
    pollWorkflowStatus() {
      // Poll frequency in milliseconds
      const poll_frequency = 5000;
      this.workflow_status_polling = setInterval(() => {
        this.executed_assets.forEach(item => {
          if (
            item.workflow_status === "" ||
            item.workflow_status === "Started" ||
            item.workflow_status === "Queued"
          ) {
            this.getWorkflowStatus(item.wf_id);
          }
        });
      }, poll_frequency);
    },
    uploadFiles() {
      let key = 'upload/' + this.file.name
      let vm = this // hate this, not sure how to get correct scope inside the progressCallback method
      console.log('creds', this.$Amplify.Auth.currentSession())
      this.$Amplify.Storage.put(key, this.file, {
        level: 'public', // not actually public in the S3 sense, this is just an amplify construct
        progressCallback(progress) {
          vm.isUploading = true
          vm.uploadValue = 0
          let uploadedDec = (progress.loaded / progress.total) * 100
          vm.uploadValue = uploadedDec
        },
      }).then(function(result) {
              vm.isUploading = null
              vm.uploadValue = null
              let location = result
              location.type = vm.file.type
              vm.runWorkflow(location)
              vm.file = null
        }
      ).catch (function(err) {
          alert(err)
          vm.isUploading = null
          vm.uploadValue = null
          vm.file = null

      });
    },
    clearHistory() {
      this.executed_assets = [];
      this.$store.commit("updateExecutedAssets", this.executed_assets);
    }
  }
};
</script>
<style>
input[type="text"] {
  width: 100%;
  padding: 12px 20px;
  margin: 8px 0;
  box-sizing: border-box;
}

label {
  font-weight: bold;
}

.note {
  color: red;
  font-family: "Courier New";
}
</style>
