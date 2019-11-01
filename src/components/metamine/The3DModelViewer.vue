<template>
  <!-- NOTE: Some code copied from https://kitware.github.io/vtk-js/examples/OBJViewer.html#Source and adapted for VueJS and this application -->
  <div class="the3dmodelviewer">
    <h1>{{ msg }}</h1>
    <v-alert
      v-model="errorAlert"
      type="error"
      dismissible
    >
      {{errorMsg}}
    </v-alert>
    <v-container fluid grid-list-md>
      <v-layout row wrap>
        <v-flex d-flex xs2>
          <div class="text-xs-left">
            View example 1: {{exampleModels[0]}}
            <v-btn class="text-xs-left" small color="primary" @click="selectExample(0)">View</v-btn>
          </div>
        </v-flex>
        <v-flex d-flex xs2>
          <div class="text-xs-left">
            View example 2: {{exampleModels[1]}}
            <v-btn class="text-xs-left" small color="primary" @click="selectExample(1)">View</v-btn>
          </div>
        </v-flex>
        <v-flex d-flex xs2>
          <div class="text-xs-left">Select an {{supportedTypes}} model file to view from your local file system. {{objfileNote}}
            <v-btn class="text-xs-left" small color="primary" @click='selectFile'>Browse</v-btn>
            <input
              type="file"
              style="display: none"
              :accept="acceptedFileTypes()"
              ref="stlfilebrowser"
              @change="onFileSelected"
            >
          </div>
        </v-flex>
      </v-layout>
      <v-flex d-flex xs12>
          <v-layout row wrap>
            <v-flex d-flex xs12 class="font-weight-black font-italic text-xs-center" v-if="modelName.length > 0">
              Model Name: {{modelName}}
            </v-flex>
            <v-flex d-flex xs12>
              <span v-if="modelName.toLowerCase().endsWith('.stl')">
                <model-stl  width="400" height="400" :src="modelData" @on-load="modelLoaded" @on-error="modelLoadError"></model-stl>
              </span>
              <span v-if="modelName.toLowerCase().endsWith('.gltf') || modelName.toLowerCase().endsWith('.glb')">
                <model-gltf  width="400" height="400" :src="modelData" @on-load="modelLoaded" @on-error="modelLoadError"></model-gltf>
              </span>
              <span v-if="modelName.toLowerCase().endsWith('.obj')">
                <model-obj  width="400" height="400" :src="modelData" @on-load="modelLoaded" @on-error="modelLoadError"></model-obj>
              </span>
              <span v-if="modelName.toLowerCase().endsWith('.vtk') || modelName.toLowerCase().endsWith('.vtb')">
                <model-vtk  width="400" height="400" :src="modelData" @on-load="modelLoaded" @on-error="modelLoadError"></model-vtk>
              </span>
            </v-flex>
          </v-layout>
        </v-flex>
    </v-container>
  </div>
</template>

<script>
/* e# slint-disable import/prefer-default-export */
/* e# slint-disable import/no-extraneous-dependencies */
import Axios from 'axios'
import JSZip from 'jszip'

import macro from 'vtk.js/Sources/macro'

import HttpDataAccessHelper from 'vtk.js/Sources/IO/Core/DataAccessHelper/HttpDataAccessHelper'
import vtkFullScreenRenderWindow from 'vtk.js/Sources/Rendering/Misc/FullScreenRenderWindow'
// import vtkURLExtract from 'vtk.js/Sources/Common/Core/URLExtract'

import vtkOBJReader from 'vtk.js/Sources/IO/Misc/OBJReader'
import vtkMTLReader from 'vtk.js/Sources/IO/Misc/MTLReader'
import vtkMapper from 'vtk.js/Sources/Rendering/Core/Mapper'
import vtkActor from 'vtk.js/Sources/Rendering/Core/Actor'

export default {
  name: 'The3DModelViewer',
  components: {HttpDataAccessHelper, vtkFullScreenRenderWindow, vtkOBJReader, vtkMTLReader, vtkMapper, vtkActor},
  data () {
    return {
      msg: '3D Model Viewer',
      errorAlert: false,
      errorMsg: '',
      modelData: null,
      modelName: '',
      currentExample: -1,
      exampleModelData: [null, null],
      exampleModels: ['latTrunCube_4921.stl', 'tpbsDD1_20.stl'],
      loaders: [
        {active: false, name: 'GLTF Model Viewer', type: 'GLTF/GLB', fileTypes: '.gltf,.glb'},
        {active: true, name: 'OBJ Model Viewer', type: 'OBJ', fileTypes: '.obj,.zip'},
        {active: true, name: 'STL Model Viewer', type: 'STL', fileTypes: '.stl'},
        {active: false, name: 'VTK Model Viewer', type: 'VTB', fileTypes: '.vtb'}
      ]
    }
  },
  computed: {
    supportedTypes () {
      let vm = this
      let rv = ''
      vm.loaders.forEach((v, idx) => {
        if (v.active) {
          if (idx >= (vm.supportedCount() - 1) && idx !== 0) {
            rv += (' or ' + v.type)
          } else {
            rv += ((rv.length > 0) ? (', ' + v.type) : v.type)
          }
        }
      })
      return rv
    },
    objfileNote () {
      let rv = ''
      let vm = this
      vm.loaders.forEach((v) => {
        if (v.active && v.type === 'OBJ') {
          rv = 'NOTE: OBJ files do not quite display - a solution is forth-coming.'
        }
      })
      return rv
    }
  },
  methods: {
    supportedCount () {
      let rv = 0
      let vm = this
      vm.loaders.forEach((v) => {
        if (v.active) {
          ++rv
        }
      })
      return rv
    },
    acceptedFileTypes () {
      let vm = this
      let accepted = ''
      vm.loaders.forEach((v) => {
        if (v.active) {
          let value = v.fileTypes
          accepted += ((accepted.length > 0) ? (',' + value) : value)
        }
      })
      return accepted
    },
    modelLoaded (ev) {
      let vm = this
      setTimeout(function () {
        vm.resetLoading()
      }, 250)
    },
    modelLoadError (ev) {
      let vm = this
      vm.resetLoading()
      vm.errorAlert = true
      vm.errorMsg = 'Error loading image.'
      vm.modelData = null
      vm.modelName = ''
      if (vm.currentExample >= 0) {
        vm.exampleModelData[vm.currentExample] = null
      }
    },
    getModelData () {
      return this.modelData
    },
    setLoading: function () {
      this.$store.commit('isLoading')
    },
    resetLoading: function () {
      this.$store.commit('notLoading')
    },
    selectExample (exampleIdx) {
      let vm = this
      vm.currentExample = exampleIdx
      console.log('exampleIdx: ' + exampleIdx)
      let fileNm = vm.exampleModels[exampleIdx]
      let fullNm = 'https://materialsmine.org/nmf/' + fileNm + '.base64'
      if (vm.exampleModelData[exampleIdx] === null) {
        vm.setLoading()
        vm.loadData(fullNm)
          .then(function (modelData) {
            // NOTE: let the model on error or on load handler reset the loading spinner
            //    vm.resetLoading()
            console.log(modelData.slice(0, 20))
            vm.exampleModelData[exampleIdx] = modelData
            vm.errorMsg = ''
            vm.errorAlert = false
            vm.modelData = vm.exampleModelData[exampleIdx]
            vm.modelName = 'Example model: ' + vm.exampleModels[exampleIdx]
          })
          .catch(function (err) {
            let msg = 'Error loading example: ' + vm.exampleModels[exampleIdx] + ' - ' + err
            console.log(msg)
            vm.errorMsg = msg
            vm.errorAlert = true
            vm.modelData = null
            vm.modelName = ''
            vm.resetLoading()
          })
      } else {
        vm.setLoading()
        vm.modelData = vm.exampleModelData[exampleIdx]
        vm.modelName = 'Example model: ' + vm.exampleModels[exampleIdx]
        setTimeout(function () {
          vm.resetLoading()
        }, 250)
      }
    },
    loadData (url) {
      return new Promise(function (resolve, reject) {
        Axios.get(url)
          .then(function (resp) {
            resolve(resp.data)
          })
          .catch(function (err) {
            reject(err)
          })
      })
    },
    selectFile () {
      let vm = this
      vm.$refs.stlfilebrowser.click()
    },
    onFileSelected (e) {
      let vm = this
      window.AppData = vm
      vm.setLoading()
      vm.currentExample = -1 // not an example
      console.log('on file selected')
      vm.modelData = null
      vm.modelName = ''
      const files = e.target.files
      let loadPromise = null
      let file = {}
      let f = files[0]
      if (f !== undefined) {
        file.fileName = f.name
        if (file.fileName.lastIndexOf('.') <= 0) {
          return
        }
        const fr = new FileReader()
        loadPromise = new Promise(function (resolve) {
          fr.addEventListener('load', function () {
            file.model = fr.result
            file.error = false
            resolve()
          })
        })
        // fr.readAsText(f)
        fr.readAsDataURL(f)
        // fr.readAsBinaryString(f)
        // vm.xmlFiles.push(file) // needs to be inside callback if callback is used
      } else {
        console.log('File Undefined and no load occurred.')
      }
      if (loadPromise) {
        loadPromise.then(function () {
          vm.modelName = file.fileName
          vm.modelData = file.model
          // console.log('model data: ' + vm.modelData)
          // let model viewer load handler resetLoading
          //     vm.resetLoading()
          vm.errorAlert = false
        })
          .catch(function (err) {
            vm.resetLoading()
            vm.errorMsg = 'Error loading file: ' + err
            vm.errorAlert = true
            // Handle error
          })
      }
    },
    iOS () {
      let rv = /iPad|iPhone|iPod/.test(window.navigator.platform)
      return rv
      // if (iOS) {
      //   document.querySelector('body').classList.add('is-ios-device');
      // }
    },
    emptyContainer (container) {
      while (container.firstChild) {
        container.removeChild(container.firstChild)
      }
    },
    loadZipContent (zipContent, renderWindow, renderer) {
      const fileContents = {obj: {}, mtl: {}, img: {}}
      const zip = new JSZip()
      zip.loadAsync(zipContent).then(() => {
        let workLoad = 0

        function done () {
          if (workLoad !== 0) {
            return
          }

          // Attach images to MTLs
          const promises = []
          Object.keys(fileContents.mtl).forEach((mtlFilePath) => {
            const mtlReader = fileContents.mtl[mtlFilePath]
            const basePath = mtlFilePath
              .split('/')
              .filter((v, i, a) => i < a.length - 1)
              .join('/')
            mtlReader.listImages().forEach((relPath) => {
              const key = basePath.length ? `${basePath}/${relPath}` : relPath
              const imgSRC = fileContents.img[key]
              if (imgSRC) {
                promises.push(mtlReader.setImageSrc(relPath, imgSRC))
                console.log('register promise')
              }
            })
          })

          Promise.all(promises).then(() => {
            console.log('load obj...')
            // Create pipeline from obj
            Object.keys(fileContents.obj).forEach((objFilePath) => {
              const mtlFilePath = objFilePath.replace(/\.obj$/, '.mtl')
              const objReader = fileContents.obj[objFilePath]
              const mtlReader = fileContents.mtl[mtlFilePath]

              const size = objReader.getNumberOfOutputPorts()
              for (let i = 0; i < size; i++) {
                const source = objReader.getOutputData(i)
                const mapper = vtkMapper.newInstance()
                const actor = vtkActor.newInstance()
                const name = source.get('name').name

                actor.setMapper(mapper)
                mapper.setInputData(source)
                renderer.addActor(actor)

                if (mtlReader && name) {
                  mtlReader.applyMaterialToActor(name, actor)
                }
              }
            })
            renderer.resetCamera()
            renderWindow.render()
          })
        }

        zip.forEach((relativePath, zipEntry) => {
          if (relativePath.match(/\.obj$/i)) {
            workLoad++
            zipEntry.async('string').then((txt) => {
              const reader = vtkOBJReader.newInstance({splitMode: 'usemtl'})
              reader.parseAsText(txt)
              fileContents.obj[relativePath] = reader
              workLoad--
              done()
            })
          }
          if (relativePath.match(/\.mtl$/i)) {
            workLoad++
            zipEntry.async('string').then((txt) => {
              const reader = vtkMTLReader.newInstance()
              reader.parseAsText(txt)
              fileContents.mtl[relativePath] = reader
              workLoad--
              done()
            })
          }
          if (relativePath.match(/\.jpg$/i) || relativePath.match(/\.png$/i)) {
            workLoad++
            zipEntry.async('base64').then((txt) => {
              const ext = relativePath.slice(-3).toLowerCase()
              fileContents.img[relativePath] = `data:image/${ext};base64,${txt}`
              workLoad--
              done()
            })
          }
        })
      })
    },
    load (container, options) {
      let vm = this
      vm.emptyContainer(container)

      const fullScreenRenderer = vtkFullScreenRenderWindow.newInstance({
        background: [0, 0, 0],
        rootContainer: container,
        containerStyle: {height: '100%', width: '100%', position: 'absolute'}
      })
      const renderer = fullScreenRenderer.getRenderer()
      const renderWindow = fullScreenRenderer.getRenderWindow()

      if (options.file) {
        if (options.ext === 'obj') {
          const reader = new FileReader()
          reader.onload = function onLoad (e) {
            const objReader = vtkOBJReader.newInstance()
            objReader.parseAsText(reader.result)
            const nbOutputs = objReader.getNumberOfOutputPorts()
            for (let idx = 0; idx < nbOutputs; idx++) {
              const source = objReader.getOutputData(idx)
              const mapper = vtkMapper.newInstance()
              const actor = vtkActor.newInstance()
              actor.setMapper(mapper)
              mapper.setInputData(source)
              renderer.addActor(actor)
            }
            renderer.resetCamera()
            renderWindow.render()
          }
          reader.readAsText(options.file)
        } else {
          vm.loadZipContent(options.file, renderWindow, renderer)
        }
      } else if (options.fileURL) {
        const progressContainer = document.createElement('div')
        // progressContainer.setAttribute('class', style.progress)
        container.appendChild(progressContainer)

        const progressCallback = (progressEvent) => {
          if (progressEvent.lengthComputable) {
            const percent = Math.floor(
              (100 * progressEvent.loaded) / progressEvent.total
            )
            progressContainer.innerHTML = `Loading ${percent}%`
          } else {
            progressContainer.innerHTML = macro.formatBytesToProperUnit(
              progressEvent.loaded
            )
          }
        }

        HttpDataAccessHelper.fetchBinary(options.fileURL, {
          progressCallback
        }).then((content) => {
          container.removeChild(progressContainer)
          vm.loadZipContent(content, renderWindow, renderer)
        })
      }
    } // ,
    // initLocalFileLoader (container) {
    //   let vm = this
    //   const exampleContainer = document.querySelector('.content')
    //   const rootBody = document.querySelector('body')
    //   const myContainer = container || exampleContainer || rootBody
    //
    //   if (myContainer !== container) {
    //     myContainer.classList.add(style.fullScreen)
    //     rootBody.style.margin = '0'
    //     rootBody.style.padding = '0'
    //   } else {
    //     rootBody.style.margin = '0'
    //     rootBody.style.padding = '0'
    //   }
    // }
  }
}
</script>

<style scoped>
  img {
    width: 240px;
  }
  h3 {
    color: #096ff4;
  }
  h4 {
    text-transform: uppercase;
  }
  h1 {
    margin-top: 10px;
    background-color: black;
    color: white;
    margin-bottom: -4px;
  }

</style>
