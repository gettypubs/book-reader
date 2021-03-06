<template>
  <div :ref="'container'"
       class="pageContainer"
       :class="containerClass"
       :style="containerStyle">
    <div class="pages" :style="pagesStyle">
      <page
        v-for="page in displayedPages"
        :key="page.pageIndex"
        :ref="'page_'+page.pageIndex"
        :page="page"
        :width="computedWidth"
        :height="height - 40"
        :scale="zoom"
        :onViewport="handleViewport"
        :onImageClicked="onImageClicked"
        :onSelected="onSelected"
        @displayed="afterDisplayed"
        class="page"
      />
    </div>
    <search :pdfDocument="pdfDocument" :query="query" @matched="this.onMatched" @found="this.onFound"/>
  </div>
</template>

<script>
import Page from '@/components/Page';
import Search from '@/components/Search';

const pdfjsLib = require('pdfjs-dist');

pdfjsLib.PDFJS.workerSrc = 'static/libs/pdfjs-dist/build/pdf.worker.min.js';

export default {
  name: 'pdf',
  components: {
    'Page': Page,
    'Search': Search
  },
  props: {
    'src': {
      type: String
    },
    'width': {
      type: Number,
      default: undefined
    },
    'height': {
      type: Number,
      default: undefined
    },
    'page': {
      default: 0,
      type: Number
    },
    'spreads': {
      default: true,
      type: Boolean
    },
    'onImageClicked': {
      default: undefined,
      type: Function
    },
    'onOutlineReady': {
      default: undefined,
      type: Function
    },
    'query': {
      default: undefined,
      type: String
    },
    'zoom': {
      default: 1,
      type: Number
    },
    'onProgress': {
      default: undefined,
      type: Function
    }
  },
  data () {
    return {
      pdfDocument: undefined,
      numPages: 0,
      scale: 1,
      rotate: undefined,
      isLoading: true,
      viewportWidth: 0,
      viewportHeight: 0,
      viewportLeft: 0,
      viewportTop: 0,
      displayedPage: 0,
      displayedPages: [],
      displayedPagesNumbers: [],
      outline: undefined,
      pageMatchesLength: [],
      pageMatches: [],
      matched: [],
      selectedMatch: {}
    }
  },
  updated () {
  },
  mounted () {
    if (this.src) {
      this.loadDocument(this.src)
        .then(() => {
          this.display(this.page);
        })
        .catch((err) => console.error(err));
    }
  },
  computed: {
    containerStyle () {
      return { width: `${this.width}px`, height: `${this.height ? this.height : this.viewportHeight}px` }
    },
    containerClass () {
      return { loading: this.isLoading, centered: (!this.spreads || this.zoom <= 1) }
    },
    pagesStyle () {
      return {
        width: `${this.viewportWidth}px`,
        height: `${this.viewportHeight}px`,
        top: `${this.viewportTop}px`,
        left: `${this.viewportLeft}px`
      }
    },
    computedWidth () {
      return this.spreads ? (this.width / 2) - 40 : this.width - 40;
    }
  },
  watch: {
    src () {
      if (this.src) {
        this.loadDocument(this.src)
          .then(() => {
            this.display(this.page);
          })
          .catch((err) => console.error(err));
      }
    },
    page (pg) {
      if (typeof this.page !== 'undefined' && this.page > -1) {
        this.display(this.page);
      }
    },
    spreads (spreads) {
      if (typeof this.displayedPage !== 'undefined') {
        this.display(this.displayedPage);
      }
    },
    displayedPage () {
      this.$emit('pageChanged', this.displayedPagesNumbers);
    },
    query () {
      if (this.query === '') {
        this.clearMatches();
      }
    },
    zoom () {
      let container = this.$refs.container;
      let bounds = container.getBoundingClientRect();
      this.scrollLeftDelta = (container.scrollLeft - ((container.scrollWidth - bounds.width) / 2));
      this.scrollTopDelta = (container.scrollTop - ((container.scrollHeight - bounds.height) / 2));
    }
  },
  methods: {
    loadDocument (src) {
      let loadingTask = pdfjsLib.getDocument(src);

      loadingTask.onProgress = (progressData) => {
        this.onProgress && this.onProgress(progressData.loaded / progressData.total)
      };

      return loadingTask.promise
        .then((pdfDocument) => {
          // Document loaded, retrieving the page.
          var pagesCount = pdfDocument.numPages;
          // var noCover = settings.cover === false;
          this.numPages = pagesCount;

          this.pdfDocument = pdfDocument;

          this.$emit('loaded', pdfDocument);

          this.outline = pdfDocument.getOutline().then((outline) => {
            return this.processOutline(outline);
          });
        })
        .catch((err) => console.error(err));
    },
    display (page = this.page) {
      this.displayedPages = []; // clear

      if (page < 0) {
        return;
      }

      if (page > this.numPages) {
        page = 0;
      }

      if (this.spreads) {
        if (page % 2 !== 0 || page === 0) {
          this.displayedPage = page;
        } else {
          this.displayedPage = page - 1;
        }

        if (this.displayedPage > 0) {
          this.loadPage(this.displayedPage) // left
          if (this.displayedPage !== this.numPages - 1) {
            this.loadPage(this.displayedPage + 1) // right
            this.displayedPagesNumbers = [this.displayedPage, this.displayedPage + 1];
          } else {
            this.displayedPagesNumbers = [this.displayedPage];
          }
        } else {
          this.loadPage(this.displayedPage) // cover
          this.displayedPagesNumbers = [this.displayedPage];
        }
      } else {
        this.displayedPage = page;
        this.loadPage(this.displayedPage);

        this.displayedPagesNumbers = [this.displayedPage];
      }
    },
    next () {
      if (!this.pdfDocument) {
        return;
      }
      if (this.displayedPage === 0) {
        this.displayedPage += 1;
      } else {
        this.displayedPage += this.spreads ? 2 : 1;
      }
      if (this.displayedPage < this.numPages) {
        this.display(this.displayedPage);
      } else {
        this.displayedPage = this.numPages - (this.spreads ? 2 : 1);
      }
    },
    prev () {
      if (!this.pdfDocument) {
        return;
      }
      if (this.displayedPage > 0) {
        if (this.displayedPage === 1) {
          this.displayedPage -= 1;
        } else {
          this.displayedPage -= this.spreads ? 2 : 1;
        }
        this.display(this.displayedPage);
      }
    },
    loadPage (pageIndex) {
      const { pdfDocument } = this;

      if (!pdfDocument) {
        throw new Error('Unexpected call to getPage() before the document has been loaded.');
      }

      let pageNumber = pageIndex + 1;

      if (!pageIndex || pageNumber < 1) {
        pageNumber = 1;
      } else if (pageNumber >= pdfDocument.numPages) {
        pageNumber = pdfDocument.numPages;
      }

      return pdfDocument.getPage(pageNumber)
        .then((page) => this.onPageLoad(page))
        .catch((page) => this.onPageError(page));
    },
    onPageLoad (page) {
      if (!this.displayedPagesNumbers.includes(page.pageIndex)) {
        return;
      }

      this.displayedPages.push(page);

      this.$nextTick(() => {
        let pg = this.$refs[`page_${page.pageIndex}`][0];
        pg.updatedSelectedMatch(this.selectedMatch);
        pg.updateTextLayerMatches(this.query, this.pageMatches, this.pageMatchesLength);
      });
    },
    afterDisplayed () {
      let container = this.$refs.container;
      let bounds = container.getBoundingClientRect();
      container.scrollLeft = (container.scrollWidth - bounds.width) / 2 + this.scrollLeftDelta;
      container.scrollTop = (container.scrollHeight - bounds.height) / 2 + this.scrollTopDelta;
    },
    onPageError (page) {
      console.error(page);
    },
    handleViewport (viewport) {
      this.viewportWidth = (this.spreads && this.displayedPage > 0 && this.displayedPage < this.numPages - 1) ? viewport.width * 2 : viewport.width;
      this.viewportHeight = viewport.height;

      let left = (this.width - this.viewportWidth) / 2;
      let top = (this.height - this.viewportHeight) / 2;

      this.viewportLeft = left > 0 ? left : 0;
      this.viewportTop = top > 0 ? top : 0;
    },
    processOutline (outline) {
      this.onOutlineReady && this.onOutlineReady(outline);
      return outline;
    },
    getPageIndex (dest) {
      return this.pdfDocument.getPageIndex(dest);
    },
    onMatched (matched, pageMatches, pageMatchesLength, count) {
      // per page
    },
    onFound (query, matched, pageMatches, pageMatchesLength, count) {
      this.pageMatches = pageMatches;
      this.pageMatchesLength = pageMatchesLength;
      this.matched = matched;
      this.$emit('found', count);
      if (this.matched && this.matched.length) {
        this.currentMatchIndex = 0;
        this.selectedMatch = this.matched[this.currentMatchIndex];
        this.$emit('match', this.currentMatchIndex);

        if (!this.displayedPagesNumbers.includes(this.selectedMatch.pageIdx)) {
          this.display(this.selectedMatch.pageIdx);
        } else {
          let pg = this.$refs[`page_${this.selectedMatch.pageIdx}`][0];
          pg.updatedSelectedMatch(this.selectedMatch);
          pg.updateTextLayerMatches(this.query, this.pageMatches, this.pageMatchesLength);
        }
      }
    },
    onSelected (selected) {
      let container = this.$refs.container;
      let bounds = container.getBoundingClientRect();
      container.scrollLeft = bounds.left + selected.left;
      container.scrollTop = bounds.top + selected.top;
    },
    clearMatches () {
      this.displayedPagesNumbers.forEach((pageIndex) => {
        let pg = this.$refs[`page_${pageIndex}`][0];
        pg.updateTextLayerMatches('', [], null);
      });
    },
    nextMatch () {
      if (!this.matched || this.matched.length === 0) {
        return;
      }

      if (this.currentMatchIndex < this.matched.length - 1) {
        this.currentMatchIndex += 1;
      } else {
        this.currentMatchIndex = 0;
      }

      this.selectedMatch = this.matched[this.currentMatchIndex];
      this.$emit('match', this.currentMatchIndex);

      if (!this.displayedPagesNumbers.includes(this.selectedMatch.pageIdx)) {
        this.display(this.selectedMatch.pageIdx);
      } else {
        let pg = this.$refs[`page_${this.selectedMatch.pageIdx}`][0];
        pg.updatedSelectedMatch(this.selectedMatch);
      }
    },
    prevMatch () {
      if (!this.matched || this.matched.length === 0) {
        return;
      }

      if (this.currentMatchIndex > 0) {
        this.currentMatchIndex -= 1;
      } else {
        this.currentMatchIndex = this.matched.length - 1;
      }

      this.selectedMatch = this.matched[this.currentMatchIndex];
      this.$emit('match', this.currentMatchIndex);

      if (!this.displayedPagesNumbers.includes(this.selectedMatch.pageIdx)) {
        this.display(this.selectedMatch.pageIdx);
      } else {
        let pg = this.$refs[`page_${this.selectedMatch.pageIdx}`][0];
        pg.updatedSelectedMatch(this.selectedMatch);
      }
    }
  }
}
</script>

<style scoped>
.pageContainer {
  overflow: scroll;
  position: relative;
}


.pages {
  position: absolute;
  top: 0;
  left: 0;

  display: flex;
  flex-direction: row;
  flex-wrap: nowrap;

  box-shadow: 0 1px 3px rgba(0,0,0,0.12), 0 1px 2px rgba(0,0,0,0.24);
}

.page {
  flex: none;
}

</style>
