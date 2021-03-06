<template>
<div id="wrapper">
  <img id="logo" src="~@/assets/logo.png" alt="electroneum-logo">
  <main>
    <div class="left-side">
      <span class="title">
          ElectroVault Installation
        </span>
      <system-information></system-information>
    </div>

    <div class="right-side">
      <div class="doc">
        <div class="title">Installing...</div>
        <ul class="install-list">
          <li v-for="step in installSteps">
            <div class="progress" v-bind:class="{ pending: step.pending, error: step.error, success: step.success }"></div> {{ step.title }}
          </li>
        </ul>
        <div v-if="installDone === true">
          <router-link class="button primary" to="/wallet" style="font-weight: 600;">Launch ElectroVault</router-link><br><br>
        </div>
        <div v-else>
          <button @click="close" class="button button-info">Cancel setup</button><br><br>
        </div>
      </div>
    </div>
  </main>
</div>
</template>

<script>
import SystemInformation from './LandingPage/SystemInformation'

var fs = require('fs');
var path = require('path');
//var cmd = require('node-cmd');
var store = require('store');
var request = require('request');
var unzip = require('unzip');

const remote = require('electron').remote

export default {
  name: 'install',
  components: {
    SystemInformation
  },
  data() {
    return {
      installSteps: [{
          'title': 'Setting up filesystem',
          'pending': false,
          'error': false,
          'success': false
        },
        {
          'title': 'Downloading Electroneum',
          'pending': false,
          'error': false,
          'success': false
        },
        {
          'title': 'Unzipping and setting some small things up',
          'pending': false,
          'error': false,
          'success': false
        },
        {
          'title': 'Finishing final touches',
          'pending': false,
          'error': false,
          'success': false
        }
      ],
      downloadsLinux: [{
        'component': 'linux-x64',
        'url': 'https://github.com/electroneum/electroneum/releases/download/v0.11.0.0/linux-x64-0.11.0.0.zip',
        'finished': false
      }, ],
      downloadsWindows: [{
          'component': 'win-x64',
          'url': 'https://github.com/electroneum/electroneum/releases/download/v0.11.0.0/win-x64-0.11.0.0.zip',
          'finished': false
        },
        {
          'component': 'win-x86',
          'url': 'https://github.com/electroneum/electroneum/releases/download/v0.11.0.0/win-x86-0.11.0.0.zip',
          'finished': false
        }
      ],
      downloadsMac: [

      ],
      complete: -1,
      installDone: false,
      w: remote.getCurrentWindow()
    }
  },
  methods: {

    // Open a link externally (Default, not really needed)
    open(link) {
      this.$electron.shell.openExternal(link);
    },

    unzipFile(file, directory) {
      fs.createReadStream(file).pipe(unzip.Extract({
        path: directory
      }));
    },

    // Closes Electron Application
    close() {
      this.w.close();
    },

    // Check if setup is complete.
    checkComplete() {
      var self = this;

      if (self.installSteps[0].success == true && self.installSteps[1].success == true && self.installSteps[2].success == true && self.installSteps[3].success == true) {
        store.set('setupComplete', true);
        self.installDone = true;
      }
    },

    // No linux or mac support as of yet.
    download(url, outputFile) {
      var self = this;
      self.installSteps[1].pending = true;

      request(url)
        .pipe(fs.createWriteStream(outputFile))
        .on('error', function(err) {
          self.installSteps[1].error = true;
          self.installSteps[1].pending = false;
        })
        .on('close', function() {
          self.installSteps[1].error = false;
          self.installSteps[1].pending = false;
          self.installSteps[1].success = true;
          self.checkComplete();
        });
    },
    startInstall() {
      var self = this;
      var electroDir = require('os').homedir() + '\\Desktop\\electrovault_wallet';
      var coreDir = require('os').homedir() + '\\Desktop\\electrovault_wallet\\core.zip';
      var walletsDir = electroDir + '\\wallets';
      var arch = require('os').arch();
      var isDownloaded = false;

      // Create Electroneum directory
      self.installSteps[0].pending = true;
      if (!fs.existsSync(electroDir)) {
        console.log("Creating main directory...");
        fs.mkdirSync(electroDir);
        if(!fs.existsSync(walletsDir)) {
          console.log("Creating wallets directory...");
          fs.mkdirSync(walletsDir);
          self.installSteps[0].success = true;
          self.checkComplete();
        } else {
          self.installSteps[0].success = true;
          self.checkComplete();
        }
        self.installSteps[0].success = true;
        self.checkComplete();
      } else if(!fs.existsSync(walletsDir)) {
        console.log("Creating wallets directory...");
        fs.mkdirSync(walletsDir);
        self.installSteps[0].success = true;
        self.checkComplete();
      } else if (fs.existsSync(electroDir) && fs.existsSync(walletsDir)) {
        console.log("Skipping directories...");
        self.installSteps[0].success = true;
      } else {
        alert('Please remove the current electrovault directory located at: "' + electroDir + '" and try again.')
        self.installSteps[0].pending = false;
        self.installSteps[0].error = true;
      }

      // Download Electroneum daemon and requisite files
      if (!fs.existsSync(coreDir)) {
        if (arch == 'x64' && self.installSteps[0].error != true) {
          var windl = self.downloadsWindows[0].url;
          console.log("Starting download...");
          self.download(windl, coreDir);
          self.downloadsWindows[0].finished = true;
          isDownloaded = true;
        } else if (arch == 'x86' || arch == 'x32' && self.installSteps[0].error != true) {
          var windl = self.downloadsWindows[1].url;
          console.log("Starting download...");
          self.download(windl, coreDir);
          self.downloadsWindows[1].finished = true;
          isDownloaded = true;
        }
      } else {
        console.log("Skipping download...")
        self.installSteps[1].error = false;
        self.installSteps[1].pending = false;
        self.installSteps[1].success = true;
        isDownload = true;
        self.checkComplete();
      }

      // Unzip Electroneum daemon and files
      // Not working when the daemon is downloaded
      if (isDownloaded) {
        console.log("Unzipping...")
        self.unzipFile(coreDir, electroDir);
        console.log("Target :: " + electroDir)
        self.installSteps[2].pending = true;
        self.installSteps[2].success = true;
        self.checkComplete();
      } else {
        self.installSteps[2].pending = false;
        self.installSteps[2].error = true;
      }

      // Last installSteps
      if (self.installSteps[2] == true) {
        alert('Last step is not complete');
      }
    }
  },
  mounted: function() {
    var self = this;
    this.startInstall()
  }
}
</script>

<style>
@import url('https://fonts.googleapis.com/css?family=Source+Sans+Pro');
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: 'Source Sans Pro', sans-serif;
}

#wrapper {
  background: radial-gradient( ellipse at top left,
  rgba(255, 255, 255, 1) 40%,
  rgba(229, 229, 229, .9) 100%);
  height: 100vh;
  padding: 60px 80px;
  width: 100vw;
}

#logo {
  float: left;
  height: 100px;
  margin-bottom: 40px;
  width: auto;
}

#logo-text {
  float: left;
  margin-left: 20px;
  line-height: 100px;
  font-size: 18pt;
  font-weight: bold;
}

main {
  clear: left;
  display: flex;
  justify-content: space-between;
}

main>div {
  flex-basis: 50%;
}

.left-side {
  display: flex;
  flex-direction: column;
}

.welcome {
  color: #555;
  font-size: 23px;
  margin-bottom: 10px;
}

.title {
  color: #2c3e50;
  font-size: 20px;
  font-weight: bold;
  margin-bottom: 6px;
}

.title.alt {
  font-size: 18px;
  margin-bottom: 10px;
}

.install-list {
  color: #555;
  font-weight: initial;
  list-style-type: none;
  margin-bottom: 50px;
}

.intall-list li {
  color: #000;
  height: 20px;
}

.install-list li code {
  color: #3e3e3e;
  font-weight: bold;
}

.progress {
  float: left;
  width: 10px;
  height: 10px;
  margin-top: 6px;
  margin-right: 5px;
  border-radius: 50%;
  background-color: #d1d1d1;
}

.pending {
  border: 2px solid #7ed35f;
  background-color: transparent;
}

.success {
  background-color: #7ed35f;
}

.error {
  background-color: #ef4049;
}

.doc .button {
  font-size: .9em;
  cursor: pointer;
  outline: none;
  padding: 0.75em 2em;
  border-radius: 2em;
  display: inline-block;
  color: #fff;
  background-color: #2F77F7;
  transition: all 0.15s ease;
  box-sizing: border-box;
  border: 1px solid #2F77F7;
  margin-top: 10px;
  text-decoration: none;
  -webkit-app-region: no-drag;
}

.doc .button:hover {
  background-color: #2262d6;
}

.doc .button-alt {
  color: #3e3e3e;
  margin-right: 5px;
  background-color: transparent;
}

.doc .button-alt:hover {
  background-color: #e2e2e2;
}

.doc .button-info {
  border-color: #3e3e3e;
  color: #3e3e3e;
  margin-right: 5px;
  background-color: transparent;
}

.doc .button-info:hover {
  background-color: #e2e2e2;
}
</style>
