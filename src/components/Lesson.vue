<template>
  <div>
    <header class="bg-navy pa3 flex items-center justify-around">
      <a href='/#/' class="dn db-ns link flex-auto w-third-ns">
        <img src="./home/ipfs-logo.svg" alt="IPFS" style="height: 66px" class="ml3-ns"/>
      </a>
      <a href='/#/' class="link ma0 ttu f3 f1-ns fw4 w-third-ns tc">
        <span class="green">Proto</span>
        <span class="white">school</span>
      </a>
      <div class="w-third-ns tr dn db-ns">
        <img src="./home/ipfs-illustrations-how-3.svg" alt="" style="height: 50px">
      </div>
    </header>

    <div class="flex-l items-start bt border-aqua bw4">
      <section class="pv3 indent-1">
        <h1 class="f3 measure-wide">{{lessonTitle}}</h1>
        <div class="lesson-text lh-copy measure-wide" v-html="parsedText"></div>
      </section>
      <section v-if="concepts" class='dn db-ns ba border-green ph4 ml3 ml5-l mt5 mb3 mr3 measure' style="background: rgba(105, 196, 205, 10%)">
        <h2 class="f5 fw2 green mt0 nb1 pt3">Useful concepts</h2>
        <div class='f6 lh-copy' v-html="parsedConcepts"></div>
      </section>
    </div>

    <section v-bind:class="{expand: expandExercise}" class="indent-1 exercise pb4 pt3 ph3 ph4-l mb5 mr5 flex flex-column" style="background: #F6F7F9;">
      <div class="flex-none">
        <h2 class="mt0 mb2 green fw4 fill-current">
          <svg viewBox="0 0 12 12" width='12' xmlns="http://www.w3.org/2000/svg" style='vertical-align:-1px'>
            <circle cx="6" cy="6" r="6"/>
          </svg>
          <span class="green ttu f6 pl2 pr3">Exercise {{lessonNumber}}</span>
          <span class="navy fw5 f5">{{lessonTitle}}</span>
          <div class="fr">
            <button
              v-if="expandExercise"
              title="go smol"
              v-on:click="toggleExpandExercise"
              class='b--transparent bg-transparent green hover-green-muted pointer focus-outline'>
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" viewBox="0 0 32 32"><path d="M16 4 L28 4 L28 16 L24 12 L20 16 L16 12 L20 8z M4 16 L8 20 L12 16 L16 20 L12 24 L16 28 L4 28z"></path></svg>
            </button>
            <button
              v-else
              v-on:click="toggleExpandExercise"
              title='embiggen the exercise'
              class='b--transparent bg-transparent charcoal-muted hover-green pointer focus-outline'>
              <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" fill="currentColor" viewBox="0 0 32 32"><path d="M16 4 L28 4 L28 16 L24 12 L20 16 L16 12 L20 8z M4 16 L8 20 L12 16 L16 20 L12 24 L16 28 L4 28z"></path></svg>
            </button>
          </div>
        </h2>
        <div v-if="exercise" v-html="parsedExercise" class='lh-copy'></div>
      </div>
      <div class="bg-white flex-auto" style='height:100%;'>
        <MonacoEditor
          class="editor"
          srcPath="."
          :height="editorHeight"
          :options="options"
          :code="code"
          theme="vs"
          language="javascript"
          @mounted="onMounted"
          @codeChange="onCodeChange">
        </MonacoEditor>
      </div>
      <div class='flex-none'>
        <div class="pv2">
          <div v-if="output.test" v-bind="output.test">
            <div class="lh-copy pv2 ph3 bg-red white" v-if="output.test.error">
              Error: {{output.test.error.message}}
            </div>
            <div class="lh-copy pv2 ph3 bg-red white" v-if="output.test.fail">
              {{output.test.fail}}
            </div>
            <div class="lh-copy pv2 ph3 bg-green white" v-if="output.test.success">
              {{output.test.success}}
              <a v-if="output.test.cid"
              class="link fw7 underline-hover dib ph2 mh2 white"  target='explore-ipld' :href='exploreIpldUrl'>
                View in IPLD Explorer
              </a>
            </div>
          </div>
          <div class="lh-copy pv2 ph3" v-else>
            Update the code to complete the exercise. Click <strong>submit</strong> to check your answer.
          </div>
        </div>
        <div class="pt3 ph2 tr">
          <div v-if="output.test && output.test.success">
            <Button v-bind:click="next" class="bg-aqua white">Next</Button>
          </div>
          <div v-else>
            <Button v-bind:click="run" class="bg-green white">Submit</Button>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<script>
import 'highlight.js/styles/github.css'
import Vue from 'vue'
import MonacoEditor from 'vue-monaco-editor'
import Explorer from './Explorer.vue'
import Button from './Button.vue'
const IPFS = require('ipfs')
const CID = require('cids')
const marked = require('marked')

const hljs = require('highlight.js/lib/highlight.js')
hljs.registerLanguage('js', require('highlight.js/lib/languages/javascript'))
hljs.registerLanguage('javascript', require('highlight.js/lib/languages/javascript'))
hljs.registerLanguage('json', require('highlight.js/lib/languages/json'))

marked.setOptions({
  highlight: code => {
    return hljs.highlightAuto(code).value
  }
})

const _eval = async (text, ipfs, modules = {}) => {
  await new Promise(resolve => ipfs.on('ready', resolve))

  let fn
  let result
  try {
    // eslint-disable-next-line
    fn = new Function('ipfs', 'require', text)
  } catch (e) {
    result = {error: e}
    return result
  }

  let require = name => {
    if (!modules[name]) throw new Error(`Cannot find modules: ${name}`)
    return modules[name]
  }
  try {
    result = await fn(ipfs, require)()
  } catch (e) {
    result = {error: e}
  }
  return result
}

const defaultCode = `/* globals ipfs */

const run = async () => {
  // your code goes here!
  // example: ipfs.dag.put({hello: 'world'})
}

return run

`

const output = {}
let oldIPFS

export default {
  components: {
    MonacoEditor,
    Explorer,
    Button
  },
  data: self => {
    return {
      text: self.$attrs.text,
      exercise: self.$attrs.exercise,
      concepts: self.$attrs.concepts,
      code: self.$attrs.code || defaultCode,
      parsedText: marked(self.$attrs.text),
      parsedExercise: marked(self.$attrs.exercise || ''),
      parsedConcepts: marked(self.$attrs.concepts || ''),
      lessonTitle: self.$attrs.lessonTitle,
      output,
      IPFS,
      expandExercise: false,
      options: {
        selectOnLineNumbers: false,
        lineNumbersMinChars: 3,
        scrollBeyondLastLine: false,
        automaticLayout: true
      }
    }
  },
  computed: {
    exploreIpldUrl: function () {
      let cid = this.output.test && this.output.test.cid && this.output.test.cid.toBaseEncodedString()
      cid = cid || ''
      return `https://ipfs.io/ipfs/QmYJETQ15RAnKXoJxqpXzcvQ2DuQA35UHwJBTjTPCSs9Ky/#/explore/${cid}`
    },
    lessonNumber: function () {
      return this.$route.path.slice(this.$route.path.lastIndexOf('/') + 1)
    },
    editorHeight: function () {
      if (this.expandExercise) {
        return undefined
      } else {
        const lineHeight = 18
        // In compact view show at least 12 lines, and at most 25 lines.
        const lines = Math.min(Math.max(this.code.split('\n').length, 12), 25)
        const height = lines * lineHeight
        return height
      }
    }
  },
  methods: {
    run: async function () {
      if (oldIPFS) {
        oldIPFS.stop()
        oldIPFS = null
      }
      let ipfs = this.createIPFS()
      let code = this.editor.getValue()
      let modules = {}
      if (this.$attrs.modules) modules = this.$attrs.modules
      let result = await _eval(code, ipfs, modules)
      if (result && result.error) {
        Vue.set(output, 'test', result)
        return
      }
      let test = await this.$attrs.validate(result, ipfs)
      Vue.set(output, 'test', test)
      if (CID.isCID(result)) {
        oldIPFS = ipfs
        Vue.set(output.test, 'cid', result)
      } else {
        ipfs.stop()
      }
    },
    createIPFS: function () {
      if (this.$attrs.createIPFS) {
        return this.$attrs.createIPFS()
      } else {
        return new IPFS({repo: Math.random().toString()})
      }
    },
    onMounted: function (editor) {
      this.editor = editor
    },
    onCodeChange: function (editor) {
      // console.log(editor.getValue())
    },
    next: function () {
      Vue.set(output, 'test', null)
      console.log(this.$route)
      let current = this.lessonNumber
      let next = (parseInt(current) + 1).toString().padStart(2, '0')
      console.log(current, next)
      this.$router.push({path: next})
    },
    toggleExpandExercise: function () {
      this.expandExercise = !this.expandExercise
    }
  }
}
</script>

<style scoped>
.editor {
  height: 100%;
  min-height: 15rem;
}
.exercise {
  overflow: hidden;
  max-width: 100%;
  width: 900px;
}
.exercise.expand {
  height: 100vh;
  width: initial;
  margin: 0;
  width: auto;
  position: fixed;
  top:0;
  left: 0;
  right:0;
}
.indent-1 {
  margin-left: 1rem;
}
@media screen and (min-width: 60rem) {
  .indent-1 {
    margin-left: 93px;
  }
}
</style>
