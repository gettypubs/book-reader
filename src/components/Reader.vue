 <template>
  <div>
    <link rel="stylesheet" href="//fonts.googleapis.com/icon?family=Material+Icons">

    <nav class="nav fixed" :class="{'open': navOpen || outlineOpen || artworkOpen}">
      <div class="nav-left">

        <a class="nav-item" @click="outlineOpen = !outlineOpen;">
          <span class="icon">
            <icon v-if="!outlineOpen" name="bars" title="Open Navigation"></icon>
            <icon v-if="outlineOpen" name="close" title="Close Navigation"></icon>
          </span>
        </a>

        <a class="nav-item" v-show="!artworkOpen && !showSearch">
          <span class="icon" @click="showSearch = !showSearch">
            <icon v-if="!showSearch" name="search" title="search"></icon>
            <icon v-if="showSearch" name="close" title="search"></icon>
          </span>
        </a>

        <div class="nav-item" v-if="showSearch && !artworkOpen">
          <div class="field has-addons">
            <p class="control has-icons-right" id="matchCount">
              <input class="input has-icons-right" name="query" v-model="query" results="5" placeholder="Search">
              <span class="icon is-small is-right" v-if="matchCount">{{currentMatchIndex}} of {{matchCount}}</span>
            </p>

            <p class="control">
              <a class="button" @click="prevMatch">
                <span class="icon is-small">
                  <icon name="chevron-left" title="Prev Match"></icon>
                </span>
              </a>
            </p>

            <p class="control">
              <a class="button" @click="nextMatch">
                <span class="icon is-small">
                  <icon name="chevron-right" title="Next Match"></icon>
                </span>
              </a>
            </p>

            <p class="control">
              <a class="button" @click="showSearch = false; query = ''">
                <span class="icon is-small">
                  <icon name="close" title="Close Search"></icon>
                </span>
              </a>
            </p>
          </div>
        </div>
      </div>

      <div class="nav-center">

        <div class="nav-item" v-if="!artworkOpen" v-show="!showSearch">
          <span v-if="!current" class="nav_title is-hidden-mobile">{{ title }}</span>
          <span v-if="!current" class="nav_title is-hidden-tablet">{{ shortTitle }}</span>
          <span v-if="current">
            <span class="nav_title is-hidden-mobile">{{ shortTitle }}</span>
            <icon class="navbar-breadcrumb is-hidden-mobile" name="caret-right"></icon>
            <span class="nav_title">{{ current }}</span>
          </span>
        </div>

        <div class="nav-item" v-if="artworkOpen">
          <span class="nav_title is-hidden-mobile">{{ shortTitle }}</span>
          <icon class="navbar-breadcrumb is-hidden-mobile" name="caret-right"></icon>
          <span>Artworks</span>
        </div>

      </div>

      <div class="nav-right">
        <a class="nav-item" @click="artworkOpen = !artworkOpen;">
          <span class="icon">
            <icon v-if="!artworkOpen" name="picture-o" title="Artwork"></icon>
            <icon v-if="artworkOpen" name="book" title="Book"></icon>
          </span>
        </a>
      </div>

    </nav>

    <section class="main" ref="main" :class="{'chrome_open': navOpen }" v-touch:tap="toggleNav">
      <progress-bar v-show="loadingValue < 100" :size="'small'" :value="loadingValue" :max="100" :show-label="false"></progress-bar>
      <PDF id="pdf" ref="pdf"
          :src="this.manifest.pdf"
          :page="page"
          :width="this.width"
          :height="this.height"
          :spreads="spreads"
          :query="query"
          :zoom="zoomLevel"
          :onImageClicked="this.onImageClicked"
          :onOutlineReady="this.onOutlineReady"
          :onProgress="this.onProgress"
          @pageChanged="this.onPageChanged"
          @found="this.onFound"
          @match="this.onMatch"
          @loaded="loaded"
          v-touch:swipe="onSwipe"
        />

      <div id="prev" class="arrow" @click="this.prev" v-show="!atStart">
        <span class="icon">
          <icon name="chevron-left" title="Prev" v-show="!outlineOpen"></icon>
        </span>
      </div>
      <div id="next" class="arrow" @click="this.next" v-show="!atEnd">
        <span class="icon">
          <icon name="chevron-right" title="Next" v-show="!outlineOpen"></icon>
        </span>
      </div>
      <div class="floater" :class="{'open': navOpen}" v-show="!artworkOpen">
        <div class="field has-addons is-pulled-left">
          <p class="control">
            <a class="button" @click="zoomOut">
              <span class="icon is-small">
                <icon name="search-minus" title="Zoom Out"></icon>
              </span>
            </a>
          </p>
          <p class="control zoom_input">
            <input class="input" type="text" :value="Math.round(zoomLevel * 100)+'%'">
          </p>
          <p class="control">
            <a class="button" @click="zoomIn">
              <span class="icon is-small">
                <icon name="search-plus" title="Zoom In"></icon>
              </span>
            </a>
          </p>
        </div>
        <div class="field has-addons is-pulled-left">
          <p class="control">
            <a class="button" :disabled="spreads ? null : 'disabled'" @click="setSpreads(false)">
              <span class="icon is-small">
                <icon name="file-o" title="Single"></icon>
              </span>
            </a>
          </p>
          <p class="control">
            <a class="button" :disabled="spreads ? 'disabled' : null" @click="setSpreads(true)">
              <span class="icon is-small">
                <icon name="columns" title="Spread"></icon>
              </span>
            </a>
          </p>
        </div>
        <div class="field has-addons is-pulled-left">
          <p class="control">
            <a class="button" @click="this.prev" :disabled="atStart ? 'disabled' : null">
              <span class="icon is-small">
                <icon name="chevron-left" title="Prev"></icon>
              </span>
            </a>
          </p>

          <p class="control page_input" v-show="!editingPage">
            <a class="button" @click="startPageEditing()">
              {{ displayedPages }} / {{ totalPages }}
            </a>
          </p>
          <p class="control page_input" v-show="editingPage">
            <input class="input" type="text" @keyup.enter="(v) => onPageEdited(v)" @keyup.escape="editingPage = false" ref="pageEditor">
          </p>
          <p class="control">
            <a class="button" @click="this.next" :disabled="atEnd ? 'disabled' : null">
              <span class="icon is-small">
                <icon name="chevron-right" title="Next"></icon>
              </span>
            </a>
          </p>

        </div>
      </div>
    </section>

    <div class="modal-background navigation_background" :class="{active: outlineOpen}" @click="outlineOpen = false;"></div>

    <section class="navigation" :class="{closed: !outlineOpen}">
      <nav class="nav">
        <div class="nav-left nav_closer">
          <a class="nav-item" @click="outlineOpen = !outlineOpen">
            <span class="icon">
              <icon v-if="outlineOpen" name="close" title="Close Navigation"></icon>
            </span>
          </a>
        </div>
      </nav>


      <div>
        <article class="media">
          <figure class="media-left">
            <p class="image is-64x64">
              <img class="cover_image" :src="manifest.cover_image_url">
            </p>
          </figure>
          <div class="media-content">
            <div class="content">
              <p>
                <strong class="media_title">{{ manifest.title }}</strong>
                <small class="media_subtitle">{{ manifest.subtitle }}</small>
                <small class="media_author">{{ manifest.author_as_it_appears }}</small>
              </p>
            </div>
          </div>
        </article>

        <outline :data="outline" :pdf="$refs.pdf" :page="displayedPage" @onClick="this.goto" @current="this.onCurrentTitle"/>

      </div>
    </section>

    <section class="artwork" v-show="artworkOpen">
      <div class="container">
        <b-tabs position="is-centered" v-model="activeTab" @change="onTabChanged">
          <b-tab-item label="Grid">
            <grid
              id="grid"
              ref="grid"
              :images="images"
              :imagesByArtwork="imagesByArtwork"
              :imagesByArtist="imagesByArtist"
              :imagesByCollection="imagesByCollection"
              :artworks="artworks"
              :artists="artists"
              :collections="collections"
              :filter="tableFilter"
              @update:filter="(val) => tableFilter = val"
              @onClick="this.onImageSelected" />
          </b-tab-item>
          <b-tab-item label="Table">
            <tablegrid
              id="table"
              ref="table"
              :images="images"
              :imagesByArtwork="imagesByArtwork"
              :imagesByArtist="imagesByArtist"
              :imagesByCollection="imagesByCollection"
              :artworks="artworks"
              :artists="artists"
              :collections="collections"
              :filter="tableFilter"
              @update:filter="(val) => tableFilter = val"
              @onImageClick="this.onImageSelected"
              @onPageClick="this.onPageSelected" />
          </b-tab-item>
        </b-tabs>
      </div>
    </section>

    <div class="modal" :class="{'is-active': modalActive}">
      <div class="modal-background" @click="modalActive = false"></div>
      <div class="modal-content">
        <detail
          :image="currentDetail"
          :manifest="manifest"
          @infoSelected="this.onInfoSelected"
          @pageSelected="this.onPageSelected"
          @closed="this.onDetailClosed"
          @displayed="this.onDetailDisplayed"/>
      </div>
      <button class="modal-close" @click="modalActive = false"></button>
    </div>

  </div>

</template>

<script>
import Vue from 'vue'
import PDF from '@/components/PDF'
import Table from '@/components/Table'
import Grid from '@/components/Grid'
import Outline from '@/components/Outline'
import Detail from '@/components/Detail'
import 'whatwg-fetch'

// Icons
import 'vue-awesome/icons/bars'
import 'vue-awesome/icons/search'
import 'vue-awesome/icons/columns'
import 'vue-awesome/icons/file-o'
import 'vue-awesome/icons/table'
import 'vue-awesome/icons/th'
import 'vue-awesome/icons/chevron-right'
import 'vue-awesome/icons/chevron-left'
import 'vue-awesome/icons/close'
import 'vue-awesome/icons/caret-right'
import 'vue-awesome/icons/plus'
import 'vue-awesome/icons/minus'
import 'vue-awesome/icons/hand-paper-o'
import 'vue-awesome/icons/i-cursor'
import 'vue-awesome/icons/picture-o'
import 'vue-awesome/icons/search-plus'
import 'vue-awesome/icons/search-minus'
import 'vue-awesome/icons/book'

import Icon from 'vue-awesome/components/Icon'
import Buefy from 'buefy';
import 'buefy/lib/buefy.css';

import debounce from 'debounce';

import Vue2TouchEvents from 'vue2-touch-events';
import ProgressBar from 'vue-bulma-progress-bar'

Vue.use(Vue2TouchEvents, {
  disableClick: true,
  touchClass: 'touched',
  tapTolerance: 10,
  swipeTolerance: 30
});

Vue.use(Buefy, {
  // defaultIconPack: 'fa'
});

export default {
  name: 'reader',
  components: {
    'PDF': PDF,
    'grid': Grid,
    'tablegrid': Table,
    'icon': Icon,
    'outline': Outline,
    'detail': Detail,
    'progress-bar': ProgressBar
  },
  props: {
    'manifest-url': {
      type: String
    },
    'page-url': {
      default: undefined
    },
    'image-url': {
      type: String
    },
    'show-grid': {
      default: undefined
    },
    'show-table': {
      default: undefined
    }
  },
  data () {
    let bounds = this.getBounds();

    return {
      manifest: {},
      images: [],
      page: 0,
      displayedPage: 0,
      spreads: true,
      manualSpreads: false,
      currentDetail: undefined,
      width: bounds.width,
      height: bounds.height,
      outline: undefined,
      outlineOpen: false,
      query: '',
      matchCount: undefined,
      currentMatchIndex: undefined,
      zoomLevel: 1.0,
      shownDetail: undefined,
      displayedDetail: undefined,
      activeTab: 0,
      title: '',
      shortTitle: '',
      current: '',
      modalActive: false,
      showSearch: false,
      artworkOpen: false,
      navOpen: false,
      totalPages: 1,
      editingPage: false,
      navTimeout: undefined,
      tableFilter: undefined,
      atStart: true,
      atEnd: true,
      loadingValue: 0,
      imagesByArtwork: {},
      imagesByArtist: {},
      imagesByCollection: {},
      artworks: [],
      artists: [],
      collections: []
    }
  },
  created () {
    if (this.pageUrl) {
      this.page = parseInt(this.pageUrl);
    }

    this.fetchManifest();

    this.currentDetail = undefined;

    this.toggleNav();

    this.addListeners();

    this.handleResize();

    this._showTable();
    this._showGrid();

    if (this.page === 0) {
      this.atStart = true;
      this.atEnd = false;
    }

    /*
    let spreads = localStorage.getItem('spreads');
    if (spreads != null) {
      this.spreads = (spreads === 'true');
    }
    */
  },
  beforeDestory () {
    this.removeListeners();
  },
  computed: {
    displayedPages () {
      if (!this.spreads || this.displayedPage === 0) {
        return this.displayedPage + 1;
      } else if (this.displayedPage === this.totalPages - 1) {
        return this.displayedPage + 1;
      } else {
        return `${this.displayedPage}–${this.displayedPage + 1}`;
      }
    }
  },
  watch: {
    showTable () {
      this._showTable();
    },
    showGrid () {
      this._showGrid();
    },
    artworkOpen (active) {
      const { grid } = this.$refs;
      grid.handleResize();
      grid.triggerLoad();
      if (!active) {
        if (typeof this.displayedPage !== 'undefined') {
          this.$router.push({ name: 'PageLink', params: { page: this.displayedPage + 1 } });
        } else {
          this.$router.push({ name: 'Manifest' });
        }
      } else {
        this.$router.push({ name: 'ArtworkLink', params: { tab: this.activeTab === 0 ? 'grid' : 'table' } });
      }
    },
    activeTab () {
      if (this.activeTab === 0) {
        this.$router.push({ name: 'ArtworkLink', params: { tab: 'grid' } });
      } else if (this.activeTab === 1) {
        this.$router.push({ name: 'ArtworkLink', params: { tab: 'table' } });
      }
    },
    pageUrl () {
      if (this.loaded) {
        let pg = parseInt(this.pageUrl) - 1;
        if (this.page !== pg && this.displayedPage !== pg) {
          this.page = pg;
        }
      }
    },
    imageUrl () {
      if (this.loaded) {
        this.currentDetail = this.imageUrl;
      }
    },
    displayedDetail () {
      if (this.displayedDetail) {
        this.$router.push({ name: 'ImageLink', params: { image: this.displayedDetail } });
      } else {
        // this.$router.push({ name: 'Manifest' });
      }
    },
    displayedPage () {
      if (typeof this.displayedPage !== 'undefined') {
        this.$router.push({ name: 'PageLink', params: { page: this.displayedPage + 1 } });
      } else {
        // this.$router.push({ name: 'Manifest' });
      }

      if (this.displayedPage === 0) {
        this.atStart = true;
      } else if (this.atStart) {
        this.atStart = false;
      }

      if (this.displayedPage === this.totalPages - (this.spreads ? 1 : 0)) {
        this.atEnd = true;
      } else if (this.atEnd) {
        this.atEnd = false;
      }
    },
    spreads () {
      // localStorage.setItem('spreads', this.spreads);
    },
    modalActive (active) {
      if (!active) {
        if (this.artworkOpen) {
          this.$router.push({ name: 'ArtworkLink', params: { tab: this.activeTab === 0 ? 'grid' : 'table' } });
        } else if (typeof this.displayedPage !== 'undefined') {
          this.$router.push({ name: 'PageLink', params: { page: this.displayedPage + 1 } });
        } else {
          this.$router.push({ name: 'Manifest' });
        }
      } else {
        if (this.displayedDetail) {
          this.$router.push({ name: 'ImageLink', params: { image: this.displayedDetail } });
        }
      }
    }
  },
  methods: {
    fetchManifest () {
      return fetch(this.manifestUrl)
        .then((response) => {
          return response.json()
        })
        .then((manifest) => {
          this.manifest = manifest;

          this.shortTitle = manifest.title;
          this.title = manifest.title;
          if (manifest.subtitle) {
            this.title += ': ' + manifest.subtitle;
          }

          this.title += ' – ' + manifest.author_as_it_appears;

          if (this.imageUrl) {
            this.currentDetail = this.imageUrl;
          }
        }).catch((err) => console.error(err));
    },
    loaded (pdfDocument) {
      this.loaded = true;

      this.pdfDocument = pdfDocument;
      // Wait for PDF to load
      this.images = this.manifest.images;

      this.images.forEach((image) => {
        if (image.artist_name) {
          let artistUri = encodeURI(image.artist_name);
          if (!this.imagesByArtist[artistUri]) {
            this.imagesByArtist[artistUri] = [];
            this.artists.push({
              uri: artistUri,
              name: image.artist_name
            });
          }
          this.imagesByArtist[artistUri].push(image);
        }

        if (image.artwork_title) {
          let artworkUri = encodeURI(image.artwork_title);
          if (!this.imagesByArtwork[artworkUri]) {
            this.imagesByArtwork[artworkUri] = [];
            this.artworks.push({
              uri: artworkUri,
              name: image.artwork_title
            });
          }
          this.imagesByArtwork[artworkUri].push(image);
        }

        if (image.collection) {
          let collectionUri = encodeURI(image.collection);
          if (!this.imagesByCollection[collectionUri]) {
            this.imagesByCollection[collectionUri] = [];
            this.collections.push({
              uri: collectionUri,
              name: image.collection
            });
          }
          this.imagesByCollection[collectionUri].push(image);
        }
      });

      this.totalPages = pdfDocument.numPages;
    },
    next (e) {
      const { pdf } = this.$refs;
      pdf.next();
      e.stopPropagation();
      e.preventDefault();
    },
    prev (e) {
      const { pdf } = this.$refs;
      pdf.prev();
      e.stopPropagation();
      e.preventDefault();
    },
    onSwipe (dir) {
      const { pdf } = this.$refs;

      if (dir === 'left') {
        pdf.next();
      } else if (dir === 'right') {
        pdf.prev();
      }
    },
    keyListener (e) {
      const { keyCode } = e;

      if (this.currentDetail || this.artworkOpen) {
        return;
      }

      if (keyCode === 37) {
        this.prev(e);
      } else if (keyCode === 39) {
        this.next(e);
      }
    },
    handleResize () {
      let bounds = this.getBounds();
      this.width = bounds.width;
      this.height = bounds.height;

      if (!this.manualSpreads) {
        if (this.width < 600 && this.spreads === true) {
          this.spreads = false;
        }

        if (this.width >= 600 && this.spreads === false) {
          this.spreads = true;
        }
      }
    },
    goto (dest) {
      const { pdf } = this.$refs;
      this.outlineOpen = false;
      this.artworkOpen = false;
      return pdf.getPageIndex(dest[0]).then((pg) => {
        this.page = pg;
      }).catch((err) => console.error(err));
    },
    onImageSelected (image) {
      // this.page = (image.page - 1);
      this.currentDetail = image;

      if (this.displayedDetail && !this.modalActive) {
        this.modalActive = true;
      }
    },
    onPageSelected (page) {
      this.page = (page - 1);
      this.modalActive = false;
      this.artworkOpen = false;
      this.currentDetail = undefined;
    },
    onInfoSelected (filter) {
      this.toggleGrid();
      this.tableFilter = filter;
      this.artworkOpen = true;
      this.modalActive = false;
      this.currentDetail = undefined;
    },
    onImageClicked (image) {
      // this.modalActive = true;
      this.currentDetail = image.objId;

      if (this.displayedDetail && !this.modalActive) {
        this.modalActive = true;
      }
    },
    onDetailClosed () {
      this.currentDetail = undefined;
      this.displayedDetail = undefined;
      this.modalActive = false;
    },
    onDetailDisplayed (detail) {
      if (detail) {
        this.displayedDetail = detail;
        this.modalActive = true;
      }
    },
    onFound (count) {
      this.matchCount = count;
    },
    onMatch (index) {
      this.currentMatchIndex = index + 1;
    },
    onCurrentTitle (title) {
      this.current = title;

      if (title === 'Cover') {
        this.current = '';
      }
    },
    getBounds () {
      let width = window.innerWidth;
      let height = window.innerHeight;
      // let header = 52;
      return {
        width: width,
        height: height
      };
    },
    setSpreads (spreads) {
      if (this.spreads !== spreads) {
        this.spreads = spreads;
      }

      this.manualSpreads = true;
    },
    onOutlineReady (outline) {
      this.outline = outline;
    },
    onPageChanged (pages) {
      this.displayedPage = pages[pages.length - 1];
      this.page = -1; // reset
    },
    onTabChanged (index) {
      const { grid, table } = this.$refs;

      if (index === 0) {
        grid.handleResize();
        grid.triggerLoad();
      }

      if (index === 1) {
        table.triggerLoad();
      }
    },
    toggleTable () {
      this.activeTab = 1;
    },
    toggleGrid () {
      this.activeTab = 0;
    },
    nextMatch () {
      const { pdf } = this.$refs;
      pdf.nextMatch();
    },
    prevMatch () {
      const { pdf } = this.$refs;
      pdf.prevMatch();
    },
    zoomIn () {
      // Handle https://en.wikipedia.org/wiki/IEEE_floating_point
      this.zoomLevel = Math.round(this.zoomLevel * 10 + 2) / 10;
    },
    zoomOut () {
      if (this.zoomLevel > 0.20) {
        this.zoomLevel = Math.round(this.zoomLevel * 10 - 2) / 10;
      }
    },
    showNav () {
      clearTimeout(this.navTimeout);
      this.navOpen = true;
    },
    hideNav () {
      this.navOpen = false;
    },
    toggleNav () {
      if (this.navOpen) {
        clearTimeout(this.navTimeout);
        this.hideNav();
      } else {
        this.showNav()
        this.navTimeout = setTimeout(() => {
          this.hideNav();
        }, 2000);
      }
    },
    onMouseMove (e) {
      let {y} = e;
      let dist = 100;
      this.mouseMoved = true;

      if (y < dist || y > window.innerHeight - dist) {
        if (!this.openNav) {
          this.openNav = setTimeout(() => {
            this.showNav();
            // this.openNav = undefined;
          }, 300);
        }
      } else {
        if (this.openNav) {
          clearTimeout(this.openNav);
          this.openNav = undefined;

          if (this.navOpen) {
            this.hideNav();
          }
        }
      }
    },
    onMouseDown (e) {
      this.mouseMoved = false;
    },
    onTouchMove (e) {
      this.mouseMoved = true;
    },
    onClick (e) {
      let {y} = e;
      let dist = 100;

      if (y > dist && y < window.innerHeight - dist) {
        if (!this.mouseMoved && !this.outlineOpen && !this.outlineOpen && !this.modalActive) {
          this.toggleNav();
        }
      }
      this.mouseMoved = false;
    },
    addListeners () {
      this._handleResize = debounce(this.handleResize.bind(this), 200);

      window.addEventListener('keyup', this.keyListener.bind(this), false);
      window.addEventListener('resize', this._handleResize, false);

      window.addEventListener('mousemove', this.onMouseMove.bind(this), false);
      window.addEventListener('mousedown', this.onMouseDown.bind(this), false);
      window.addEventListener('click', this.onClick.bind(this), false);
    },
    removeListeners () {
      window.removeEventListener('resize', this.handleResize);
      window.removeEventListener('keyup', this.keyListener);

      window.removeEventListener('mousemove', this.onMouseMove);
      window.removeEventListener('mousedown', this.onMouseDown);
      window.removeEventListener('click', this.onClick);
    },
    startPageEditing () {
      this.editingPage = true;
      this.$refs.pageEditor.focus();
      this.$refs.pageEditor.value = '';
    },
    onPageEdited (v) {
      let value = parseInt(v.target.value);
      if (value) {
        this.page = value - 1;
      }
      this.editingPage = false;
    },
    onProgress (progress) {
      let val = progress * 100;
      this.loadingValue = val;
    },
    _showTable () {
      const { table } = this.$refs;

      if (this.showTable) {
        this.artworkOpen = true;
        this.toggleTable();
      }

      table && table.triggerLoad();
    },
    _showGrid () {
      const { grid } = this.$refs;

      if (this.showGrid) {
        this.artworkOpen = true;
        this.toggleGrid();
      }

      grid && grid.triggerLoad();
    },
  }
}
</script>

<style lang="scss">
  // Import Bulma's core
  @import "~bulma/sass/utilities/_all";
  // Set your colors
  $primary: #1565c0;
  $primary-invert: findColorInvert($primary);
  // Setup $colors to use as bulma classes (e.g. 'is-twitter')
  $colors: (
      "white": ($white, $black),
      "black": ($black, $white),
      "light": ($light, $light-invert),
      "dark": ($dark, $dark-invert),
      "primary": ($primary, $primary-invert),
      "info": ($info, $info-invert),
      "success": ($success, $success-invert),
      "warning": ($warning, $warning-invert),
      "danger": ($danger, $danger-invert)
  );
  // Links
  $link: $primary;
  $link-invert: $primary-invert;
  $link-focus-border: $primary;
  // Import Bulma and Buefy styles
  @import "~bulma";
  @import "~buefy/src/scss/buefy";
</style>

<style>

.nav.fixed {
  top: -60px;
  left: 0;
  width: 100%;
  border-bottom: 1px solid rgba(219, 219, 219, 0.85);
  z-index: 15;
  background-color: rgba(255, 255, 255, 0.85);
  position: fixed;
  transition: top .25s ease-in;
}

.nav.fixed.open {
  top: 0;
  left: 0;
}

.nav.fixed .icon {
  /*color: #4a4a4a;*/
}

.nav.section_nav {
  background-color: whitesmoke;

}

.artwork {
  position: absolute;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  margin-bottom: 0;
  padding: 60px 40px 0 40px;
  background-color: whitesmoke;
  overflow: auto;
}

.navigation {
  position: absolute;
  top: 0;
  left: 0;
  width: 400px;
  height: 100vh;
  margin-bottom: 0;
  /*padding: 0 40px 0 40px;*/
  background-color: white;
  overflow: hidden;
  overflow-y: auto;
  z-index: 40;
  transition: left .25s ease-in;
  box-shadow: 0 3px 6px rgba(0,0,0,0.16), 0 3px 6px rgba(0,0,0,0.23);
}

.navigation_background {
  opacity: 0;
  transition: opacity .25s ease-in;
  z-index: -1;
}

.navigation_background.active {
  opacity: 1;
  z-index: 39;
}

.navigation.closed {
  left: -400px;
}

.navigation .nav .nav-right {
  flex-grow: 2;
}

.media {
  /*background: #eee;*/
  padding: 0 52px 20px 52px;
}

.media_title {
  display: block;
}

.media_subtitle {
  display: block;
  line-height: 1em;
}

.media_author {
  display: block;
  margin-top: 1em;
}

.main {
  background-color: whitesmoke;
  width: 100%;
  overflow: hidden;
  position: relative;
  -webkit-tap-highlight-color: rgba(0,0,0,0);
  -webkit-tap-highlight-color: transparent; /* For some Androids */
}

.icon.is-small svg {
  height: 1em;
}

.zoom_input {
  width: 60px;
}

.zoom_input .input {
  text-align: center;
}


.page_input {
  width: auto;
}

.page_input input {
  width: 60px;
}
.contents {
  padding: 40px;
}

.cover_image {
  max-height: 100px;
  max-width: 100px;
}

.arrow {
  position: absolute;
  top: 50vh;
  margin-top: -24px;
  /*font-size: 64px;*/
  /*color: #E2E2E2;*/
  color: rgb(74, 74, 74);
  cursor: pointer;
  -webkit-user-select: none;
  -moz-user-select: none;
  user-select: none;
  background-color: whitesmoke;
  opacity: .5;
  padding: 4px;
}

.arrow .icon {
  width: 30px;
}

.arrow:hover {
  /*color: #777;*/
}

#prev.arrow:active .icon {
  justify-content: flex-start;
}

#next.arrow:active .icon {
  justify-content: flex-end;
}

#prev {
  left: 0px;
  border-radius: 0 2px 2px 0;
}

#next {
  right: 0px;
  border-radius: 2px 0 0 2px;
}

#detail {
  z-index: 50;
}

#outline, #artwork {
  position: absolute;
  top: 60px;
  left: 20px;
  width: calc(100vw - 120px);
  height: calc(100vh - 160px);
  padding: 40px;
  background: #333333;
  box-sizing: content-box;
  -webkit-box-sizing: content-box;
  -moz-box-sizing: content-box;
  overflow: auto;
}

#outline h2 {
  display: block;
  color: #fff;
}

#outline ul {
  list-style: none;
}

#outline a.outline_link {
  color: #eee;
}

.navbar-breadcrumb {
  margin: 0 0.7em;
  height: .7em;
}

.floater {
  position: absolute;
  bottom: -60px;
  left: 0;
  position: fixed;
  transition: bottom .25s ease-in;
  color: whitesmoke;
  border-radius: 5px;
  width: 100%;
  display: flex;
  justify-content: center;
}

.floater.open {
  bottom: 20px;
}

.floater .field {
  margin: 0 4px;
  box-shadow: 0 3px 6px rgba(0,0,0,0.16), 0 3px 6px rgba(0,0,0,0.23);
  margin-bottom: 0.75rem;
  height: auto;
}

.modal-content {
  width: auto !important;
}

.modal-close {
  min-width: 48px;
  min-height: 48px;
}

#matchCount {
  align-items: flex-end;
}

#matchCount span {
  width: 5.25em;
}

.progress-container {
  position: fixed;
  top: 0;
  margin-bottom: 0;
  width: 100vw;
}

.main .progress, .main .progress.is-small {
  height: 2px;
  border-radius: 0;
}

@media screen and (max-width: 400px) {

  .nav-item {
    padding: 0.5rem 0.5rem;
    font-size: 0.9rem;
  }

  .arrow {
    transition: right .25s ease-in, left .25s ease-in;
  }

  #next {
    right: -38px;
  }

  #prev {
    left: -38px;
  }

  .chrome_open #next {
    right: 0;
  }

  .chrome_open #prev {
    left: 0;
  }

  .nav-center {
    flex: 100;
  }

  .nav_title {
    margin-left: -34px;
    font-size: 0.9rem;
  }

  .modal-content {
    max-height: calc(100vh - 80px);
    margin-bottom: 40px;
  }

  .modal .modal-close {
    top: auto;
    bottom: 12px;
    left: 50%;
    margin-left: -24px;
    min-height: 40px;
    min-width: 40px;
  }

  .floater .page_input .button,
  .floater .zoom_input .input {
    font-size: 0.8rem;
    padding-top: .45rem;
    padding-bottom: .45rem;
    height: auto;
    display: inline-block;
  }

  .zoom_input {
    width: 50px;
  }
}
</style>
