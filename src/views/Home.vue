<template>
  <v-container fluid @dragover.prevent>
    <v-row>
      <v-col align="center">
        <v-card @drop.prevent="handleFileDrop" height="400px" width="400px">
          <v-card-title class='justify-center'>
            Drop files here!
          </v-card-title>
          <v-card-text>
            <div v-if='file'>
              <v-span>
                {{file.name}}
              </v-span>
              <br/>
              <v-span v-if='currentPage != totalPages'>
                {{currentPage}} / {{totalPages}}
              </v-span>
              <v-span v-else>
                Done processing {{totalPages}} pages!
              </v-span>
            </div>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
    <v-row>
      <v-col align="right">
        <v-btn :disabled="!letterBytes" color='primary' @click='downloadFile(letterBytes, `${getBasename(file.name)}_Letter.pdf`)'>
          Letter
        </v-btn>
      </v-col>
      <v-col align="left">
        <v-btn :disabled="!legalBytes" color='primary' @click='downloadFile(legalBytes, `${getBasename(file.name)}_Legal.pdf`)'>
          Legal
        </v-btn>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>
import { PDFDocument } from 'pdf-lib'

const LETTER_RATIO = 8.5 / 11;
const LEGAL_RATIO = 8.5 / 14;
const MIDPOINT = (LETTER_RATIO + LEGAL_RATIO) / 2;

export default {
  name: 'Home',
  data() {
    return {
      file: null,
      totalPages: null,
      currentPage: null,
      letterBytes: null,
      legalBytes: null,
    }
  },
  watch: {
    file() {
      console.log(this.file);
      this.split();
    }
  },
  methods: {
    getBasename(str) {
      return str.split('.').slice(0, -1).join('.')
    },
    handleFileDrop(e) {
      let file = e.dataTransfer.files[0];

      if(!file) return;
      
      this.file = file;
      this.currentPage = null;
      this.totalPages = null;
      this.letterBytes = null;
      this.legalBytes = null;
    },
    async split() {
      if(!this.file) return;
      const originalBytes = await this.file.arrayBuffer();
      const pdf = await PDFDocument.load(originalBytes);
      const pages = pdf.getPages();

      const letterDoc = await PDFDocument.create();
      const legalDoc = await PDFDocument.create();
      
      let letterDocIndices = [];
      let legalDocIndices = [];

      this.totalPages = pages.length;
      
      for(const [i, page] of pages.entries()) {
        if(this.isLetter(page)) {
          letterDocIndices.push(i);
        } else {
          legalDocIndices.push(i);
        }
        this.currentPage = i + 1;
      }

      const letterPages = await letterDoc.copyPages(pdf, letterDocIndices);
      const legalPages = await legalDoc.copyPages(pdf, legalDocIndices);

      for(const [i, letterPage] of letterPages.entries()) {
        letterDoc.insertPage(i, letterPage);
      }
      
      for(const [i, legalPage] of legalPages.entries()) {
        legalDoc.insertPage(i, legalPage);
      }

      this.letterBytes = letterDocIndices.length ? await letterDoc.save() : null;
      this.legalBytes = legalDocIndices.length ? await legalDoc.save() : null;

      const basename = this.file.name.split('.').slice(0, -1).join('.')
      this.downloadFile(this.letterBytes, `${basename}_Letter.pdf`);
      this.downloadFile(this.legalBytes, `${basename}_Legal.pdf`);
    },
    isLetter(page) {
      console.log(page)
      const { width, height } = page.getSize();
      const ratio = width / height;
      return ratio >= MIDPOINT;
    },
    downloadFile(bytes, filename) {
      const blob = new Blob([bytes], { type: 'application/pdf' });
      const link = document.createElement('a');
      link.href = window.URL.createObjectURL(blob);
      link.download = filename;
      link.click();
    }
  },
  components: {
  }
}
</script>
