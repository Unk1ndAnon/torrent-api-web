<template>
  <v-layout
    column
    justify-sm-end
    align-center
  >
    <v-row no-gutters>
      <v-flex
        xs10
        sm10
        md6
      >
        <v-text-field
          v-model="form.keyword"
          label="search movie"
          dense
          solo
          rounded
          append-icon="search"
          @click:append="search"
        ></v-text-field>

      </v-flex>

      <v-flex xs1>
        <v-select
          v-model="form.per_page"
          :items="per_page"
          :value="form.per_page"
          dense
        ></v-select>
      </v-flex>
      <!-- <v-flex xs2>
        <v-select
          v-model="form.provider"
          :items="providers"
          item-text="name"
          item-value="name"
          :value="providers.length && providers[0].name"
          dense
        ></v-select>
      </v-flex> -->
      <!-- <v-flex xs1>
        <v-btn @click="search">search</v-btn>
      </v-flex> -->
    </v-row>
    <v-row>
      <v-flex
        xs10
        sm10
        md4
        v-if="movieInfo"
      >
        <MovieCard :movie="movieInfo" />
      </v-flex>
      <v-flex
        xs10
        sm10
        md8
        v-if="movies"
        style="padding-top:48px !important"
      >

        <v-data-table
          :headers="headers"
          :items="movies"
          hide-default-footer
          class="elevation-1"
          style="min-height:500px"
        >

          <template v-slot:item.actions="{item}">
            <div class="floatbox">
              <v-btn
                text
                icon
                @click="displayMagnet(item.magnet)"
              >
                <v-icon>museum</v-icon>
              </v-btn>
              <v-btn
                text
                icon
                @click="remoteDownload(item.magnet)"
              >
                <v-icon>cloud</v-icon>
              </v-btn>
            </div>
          </template>
        </v-data-table>
      </v-flex>
    </v-row>
  </v-layout>
</template>

<script>
import WebTorrent from 'webtorrent-hybrid'
import MovieCard from '@/components/MovieCard'
const WT = new WebTorrent()
const API_URL = process.env.API_URL

export default {
  components: {
    MovieCard
  },
  name: 'movieIndex',
  props: ['inKeyword'],
  data () {
    return {
      keyword: '',
      providers: [],
      per_page: ['1', '5', '10', '20'],
      form: {
        keyword: '',
        per_page: '5',
      },
      movies: null,
      active: null,
      searchResults: [],
      loading: false,
      movieMP4: null,
      file: null,
      providers: [],
      downloadedFile: {},
      WebTorrent: WT,
      headers: [
        {
          text: 'Title',
          align: 'left',
          sortable: false,
          value: 'title',
        },
        {
          text: 'size',
          align: 'left',
          sortable: false,
          value: 'size',
        },
        {
          text: 'Provider',
          align: 'left',
          sortable: false,
          value: 'provider',
        },
        {
          text: 'Time',
          align: 'left',
          sortable: false,
          value: 'time',
        },
        // {
        //   text: 'Magnet',
        //   align: 'left',
        //   sortable: false,
        //   value: 'magnet',
        // },
        {
          text: '',
          align: 'left',
          sortable: false,
          value: 'actions',
        }
      ],
      movieInfo: null
    };
  },
  computed: {
    magnet () {
      console.log(this.active, this.movies)
      if (this.active != null && this.movies && this.movies.length) {
        return this.movies[this.active].magnet
      }
      return null
    },
  },
  async mounted () {
    if (this.inKeyword) {
      this.form.keyword = this.inKeyword
      this.search()
    }
    // const { data } = await this.$axios.get(`${API_URL}/torrent/activeProviders`)
    // this.providers = data
  },

  methods: {
    async remoteDownload (torrentId) {
      this.loading = true
      try {
        const params = {
          torrentId,
        }
        const { data } = await this.$axios.post(`${API_URL}/torrent/download`, { params })

        this.searchResults = data
        if (data && data.torrent_count) {
          this.movies = data.movies
        }
      }
      catch (err) {
        console.log(err)
      } finally {
        this.loading = false
      }
    },
    async pickMagnet ({ index, movie }) {
      console.log(movie)
      const magnet = movie.hasOwnProperty('magnet') ? movie.magnet.replace('magnet:?', '') : ''
      this.$router.push(`/movie/play/${encodeURIComponent(movie.magnet)}`)
    },
    async downloadTorrent (torrentId) {
      // Download the torrent
      console.log(torrentId)
      if (this.downloadedFile.hasOwnProperty(torrentId)) {
        this.downloadedFile[torrentId].file.renderTo('video#player')
      } else {
        console.log('a')
        WT.add(torrentId, function (torrent) {
          // Torrents can contain many files. Let's use the .mp4 file
          console.log('bn', torrent)
          var file = torrent.files.find(function (file) {
            console.log(file.name)
            return file.name.endsWith('.mp4') || file.name.endsWith('.avi') || file.name.endsWith('.mpeg')
          })
          console.log('c', file)
          // Display the file by adding it to the DOM.
          // Supports video, audio, image files, and more!
          file.appendTo('#test')
        })
      }
    },
    async search () {
      this.loading = true
      try {
        const params = {
          ...this.form,
        }

        const movieInfo = await this.$axios.get(`${API_URL}/movie`, { params })
        console.log('movieInfo', movieInfo)
        this.movieInfo = movieInfo.data
        const { data } = await this.$axios.get(`${API_URL}/torrent`, { params })
        console.log(data)

        this.searchResults = data
        if (data && data.torrent_count) {
          this.movies = data.movies
        }
      }
      catch (err) {
        console.log(err)
      } finally {
        this.loading = false
      }
    }

  },
}
</script>
<style scoped lang="scss">
.row {
  width: 100%;
}
.floatbox {
  width: 100px;
  .v-btn {
    float: left;
  }
}
</style>