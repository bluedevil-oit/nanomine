<!--
################################################################################
#
# File Name: CorrelationReconstruct.vue
# Application: templates
# Description:
#
# Created by: Akshay Iyer, December 1, 2018
# Customized for NanoMine
#
################################################################################
-->

<template>
  <div class="CorrelationReconstruct">
    <h1>{{ msg }}</h1>
    <v-container class="text-xs-left">
      <v-flex xs12 >
          <h3>Description</h3>
          <br>
            <p>Upload a binarized image / ZIP file containing set of images (Supported file formats: .jpg, .tif, .png) and click "Reconstruct" to reconstruct statistically equivalent image.
            All correlation functions are evaluated for the "white" phase in image.</p>
            <h4> Input Options:</h4>
                  <p><strong> NOTE: Images must be binarized.</strong></p>
                  <p><strong> 1) Single image: </strong>Supported image formats are .jpg, .tif and .png.(Download <a href='http://129.105.90.149/nm/Two_pt_MCR/Input Samples/Option1.zip'>Sample</a>)</p>
                  <p><strong> 2) Single image in .mat format :</strong> The .mat file must contain ONLY ONE variable named "Input" - which contains pixel values(only 0 or 1)(Download <a href='http://129.105.90.149/nm/Two_pt_MCR/Input Samples/Option2.zip'>Sample</a>).</p>
                  <p><strong> 3) Multiple images in ZIP File:</strong> Submit a ZIP file containing multiple images (supported formats: .jpg, .tif, .png) of same size (in pixels). DO NOT ZIP the folder containing images; select all images and ZIP them directly.(Download <a href='http://129.105.90.149/nm/Two_pt_MCR/Input Samples/Option3.zip'>Sample</a>).
                  <strong> The mean value of chosen correlation (averaged over all images) will be used for reconstruction.</strong> </p>
      </v-flex>
      <v-alert
        v-model="loginRequired"
        type="error"
        outline
      >
        {{loginRequiredMsg}}
      </v-alert>
      <v-alert
        v-model="errorAlert"
        type="error"
        dismissible
      >
        {{errorAlertMsg}}
      </v-alert>
      <v-dialog v-model="successDlg" persistent max-width="500px">
        <v-card>
          <v-card-title>
            <span>Characterization Job Submitted Successfully</span>
            <v-spacer></v-spacer>
          </v-card-title>
          <v-card-text>
            Your characterization job is: {{jobId}} <br/> You should receive an email with a link to the job output.
          </v-card-text>
          <v-card-actions>
            <v-btn color="primary" flat @click="successDlgClicked()">Close</v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <v-select label="Correlation Name" v-bind:items="corr_options" v-model="corr"></v-select>
      <v-select label="Number Of Reconstructions" v-bind:items="options" v-model="num_recon"></v-select>
      <v-flex xs12 class="text-xs-center text-sm-center text-md-center text-lg-center">
        <p class="text-xs-left">Select File
          <v-btn class="text-xs-left" small color="primary" @click='pickFile'>Browse</v-btn>
          <input
            type="file"
            style="display: none"
            accept=".jpg, .png, .tif, .mat, .zip"
            ref="myUpload"
            @change="onFilePicked"
          >
        </p>
        <v-list v-model="fileName" subheader: true v-if="fileUploaded">
          <v-list-tile
            v-for="file in filesDisplay"
            :key="file.fileName"
          >
            <v-list-tile-avatar>
              <v-icon color="primary">check_circle_outline</v-icon>
            </v-list-tile-avatar>
            <v-list-tile-content>
              <v-list-tile-title v-text="file.fileName"></v-list-tile-title>
            </v-list-tile-content>
          </v-list-tile>
        </v-list>
      </v-flex>
      <v-flex class="text-xs-center">
      <v-btn v-on:click="submit()" color="primary">Reconstruct</v-btn>
      </v-flex>
      <h4>References</h4>
      <v-flex xs12>
        <p>Rintoul, M.D. and Torquato, S., 1997. Reconstruction of the structure of dispersions. <i>Journal of Colloid and Interface Science</i>, 186(2), pp.467-476.</p>
        <p>Yeong,C. and Torquato,S., 1998. Reconstructing random media Physical Review E, vol. 57, no. 1, p. 495</p>
      </v-flex>
    </v-container>
  </div>
</template>

<script>
import {} from 'vuex'
import {JobMgr} from '@/modules/JobMgr.js'
import {Auth} from '@/modules/Auth.js'

export default {
  name: 'CorrelationReconstruct',
  data: () => {
    return ({
      title: 'Input Upload',
      msg: 'Microstructure Reconstruction - Correlation Function Approach',
      dialog: false,
      fileName: '',
      // file_type: [],
      files: [],
      filesDisplay: [],
      errorAlert: false,
      errorAlertMsg: '',
      loginRequired: false,
      loginRequiredMsg: '',
      fileUploaded: false,
      successDlg: false,
      jobId: '',
      num_recon: null,
      corr: null,
      options: [
        '1', '2'
      ],
      corr_options: [
        'Autocorrelation', 'Lineal Path Correlation', 'Cluster Correlation', 'Surface Correlation'
      ]
    })
  },
  beforeMount: function () {
    let vm = this
    vm.auth = new Auth()
    if (!vm.auth.isLoggedIn()) {
      vm.loginRequired = true
      vm.loginRequiredMsg = 'Login is required.'
    }
  },
  methods: {
    setLoading: function () {
      this.$store.commit('isLoading')
    },

    resetLoading: function () {
      this.$store.commit('notLoading')
    },

    pickFile () {
      this.$refs.myUpload.click()
    },

    resetFiles: function () {
      this.files = []
      this.filesDisplay = []
      this.fileUploaded = false
    },

    onFilePicked (e) {
      this.resetFiles()
      const files = e.target.files
      for (let i = 0; i < files.length; i++) {
        let file = {}
        let f = files[i]
        if (f !== undefined) {
          file.fileName = f.name
          if (file.fileName.lastIndexOf('.') <= 0) {
            return
          }
          console.log(file.fileName)
          const fr = new FileReader()
          fr.readAsDataURL(f)
          fr.addEventListener('load', () => {
            file.fileUrl = fr.result
            this.files.push(file)
            this.filesDisplay.push(file)
            this.fileUploaded = true
          })
        } else {
          console.log('File Undefined')
        }
      }
    },
    successDlgClicked: function () {
      let vm = this
      console.log('Success dlg button clicked')
      vm.$router.go(-2) // go back to mcr homepage page
    },
    submit: function () {
      let vm = this
      vm.files.forEach(function (v) {
        console.log(JSON.stringify(v))
      })

      vm.setLoading()
      console.log('Loading..')
      let jm = new JobMgr()
      console.log('Called Job Manager')
      jm.setJobType('CorrelationReconstruct')
      jm.setJobParameters({'CorrelationType': vm.corr, 'NumOfReconstructs': vm.num_recon, 'InputType': vm.fileName.split('.').pop()}) // Figure out which input type
      if (vm.files && vm.files.length >= 1) {
        vm.files.forEach(function (v) {
          jm.addInputFile(v.fileName, v.fileUrl)
          console.log('Job Manager added file: ' + v.fileName)
        })
        return jm.submitJob(function (jobId) {
          console.log('Success! JobId is: ' + jobId)
          vm.jobId = jobId
          vm.resetLoading()
          vm.successDlg = true
        }, function (errCode, errMsg) {
          let msg = 'error: ' + errCode + ' msg: ' + errMsg
          console.log(msg)
          vm.errorAlertMsg = msg
          vm.errorAlert = true
          vm.resetLoading()
        })
      } else {
        let msg = 'Please select a file to process.'
        vm.errorAlertMsg = msg
        vm.errorAlert = true
        vm.resetLoading()
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  img {
    width: 240px;
  }

  h4 {
    text-transform: uppercase;
  }
  h1 {
    margin-top: 10px;
    background-color: black;
    color: white;
  }

</style>
