<template>
  <section class="movie">
    <div class="movie__container" v-if="movieLoaded">
      <header class="movie__header" :class="{'movie__header--page': type=='page'}" :style="{ 'background-image': 'url(' + movieBackdropSrc + ')' }">
        <div class="movie__wrap movie__wrap--header" :class="{'movie__wrap--page': type=='page'}">
          <figure class="movie__poster">
            <img v-if="moviePosterSrc" class="movie__img" src="~assets/placeholder.png" v-img="moviePosterSrc">
            <img v-if="!moviePosterSrc" class="movies-item__img is-loaded" src="~assets/no-image.png">
          </figure>
          <div class="movie__title">
            <h1 class="movie__title-text">
              {{ movie.title }}
              <!-- <span>{{ movie.type }}</span> -->
            </h1>
            <span>
            </span>
          </div>
        </div>
      </header>
      <div class="movie__main">
        <div class="movie__wrap movie__wrap--main" :class="{'movie__wrap--page': type=='page'}">
          <!-- <div class="movie__ratings">
            <p>here</p>
          </div> -->
          <!-- <div class="movie__actions" v-if="userLoggedIn && favoriteChecked"> -->
          <div class="movie__actions">

            <a class="movie__actions-link" v-if="matched" :class="{'active' : matched}">
              <svg class="movie__actions-icon">
                <use xlink:href="#iconExsits"></use>
              </svg>
              <span class="movie__actions-text"> Already in plex &nbsp;🎉</span>
            </a>
            <a class="movie__actions-link" v-else="matched">
              <svg class="movie__actions-icon">
                <use xlink:href="#iconNot_exsits"></use>
              </svg>
              <span class="movie__actions-text"> Not in plex yet</span>
            </a>

            <a class="movie__actions-link" :class="{'active' : requested}" v-if="this.requested">
              <svg class="movie__actions-icon">
                <use xlink:href="#iconSent"></use>
              </svg>
              <span class="movie__actions-text"> Requested to be downloaded</span>
            </a>
            <a class="movie__actions-link" v-else="this.requested"  @click.prevent="sendRequest">
              <svg class="movie__actions-icon" :class="{'waiting' : requested}">
                <use xlink:href="#iconUnmatched"></use>
              </svg>
              <span class="movie__actions-text"> Request to be downloaded?</span>
            </a>

            <a class="movie__actions-link" @click="showTorrents=true" v-if="admin==='true'" :class="{'active' : showTorrents}">
              <svg class="movie__actions-icon">
                <use xlink:href="#icon_torrents"></use>
              </svg>
              <span class="movie__actions-text"> Search for torrents</span>
            </a>

            <a class="movie__actions-link" @click.prevent="openTmdb">
              <svg class="movie__actions-icon">
                <use xlink:href="#icon_info"></use>
              </svg>
              <span class="movie__actions-text"> See more info</span>
            </a>
          </div>
          <div class="movie__info">
            <div v-if="movie.summary" class="movie__description">
              {{ movie.summary }}
            </div>
            <div class="movie__details">
             <!--  <div v-if="movie.genres.length" class="movie__details-block">
                <h2 class="movie__details-title">
                  Genres
                </h2>
                <div class="movie__details-text">
                  {{ nestedDataToString(movie.genres) }}
                </div>
              </div> -->
              <div v-if="movie.year" class="movie__details-block">
                <h2 class="movie__details-title">Release Date</h2>
                <div class="movie__details-text">{{ movie.year }}</div>
              </div>

               <!-- <div v-if="movie.score" class="movie__details-block">
                <h2 class="movie__details-title">Rating</h2>
                <div class="movie__details-text">{{ rating }}</div>
              </div> -->

              <div v-if="movie.type == 'show'" class="movie__details-block">
                <h2 class="movie__details-title">Seasons</h2>
                <div class="movie__details-text">{{ movie.seasons }}</div>
              </div>
            </div>
          </div>

          <!-- <TableDemo class="movie__admin">This is it</TableDemo> -->
        
          <div class="movie__admin" v-if="admin == 'true' && showTorrents">
              <h2 class="movie__admin-title">torrents: {{ movie.title }}</h2>
              <TorrentList :query="movie.title" :tmdb_id="movie.id"></TorrentList>
          </div>
        </div>
      </div>
    </div>
  </section>
</template>

<script>
import axios from 'axios'
import storage from '../storage.js'
import img from '../directives/v-image.js'
import formatDate from '../directives/v-formatDate.js'
import TorrentList from './TorrentList.vue'

export default {
  props: ['id', 'type', 'mediaType'],
  components: { TorrentList },
  directives: {
    img: img,
    formatDate: formatDate
  },
  data(){
    return{
      movie: {},
      movieLoaded: false,
      moviePosterSrc: '',
      movieBackdropSrc: '',
      userLoggedIn: storage.sessionId ? true : false,
      favoriteChecked: false,
      requested: false,
      admin: localStorage.getItem('admin'),
      showTorrents: false
    }
  },
  computed: {
    loaded(){
      return this.movieLoaded ? true : false;
    }
  },
  methods: {
    fetchMovie(id){
      this.id = id;
      (this.mediaType == 'show') ? this.tmdbType = 'show' : this.tmdbType = 'movie'
      axios.get(`https://api.kevinmidboe.com/api/v1/plex/request/${id}?type=${this.mediaType}`)
      .then(function(resp){
          let movie = resp.data;
          this.movie = movie;
          this.poster();
          this.backdrop();
          this.matched = this.movie.matchedInPlex;
          this.requested = this.movie.requested;
          if(this.userLoggedIn){
            this.checkIfInFavorites(movie.id);
            this.movieLoaded = true;
          } else {
            this.movieLoaded = true;
          }
          // Push state
          if(storage.createMoviePopup){
            storage.moviePath = this.mediaType + '/' + id;
            history.pushState({ popup: true }, null, storage.moviePath);
            storage.createMoviePopup = false;
          }
          // Change Page title
          document.title = this.movie.title + storage.pageTitlePostfix;
      }.bind(this))
      .catch(function(error) {
        console.log(error.response)
        this.$router.push({ name: '404' });
      }.bind(this));
    },
    poster() {
      // Change the poster resolution
      if(this.movie.poster_path){
        this.moviePosterSrc = 'https://image.tmdb.org/t/p/w300' + this.movie.poster_path;
      }
    },
    backdrop(){
      if(this.movie.background_path){
        this.movieBackdropSrc = 'https://image.tmdb.org/t/p/w500' + this.movie.background_path;
      }
    },
    nestedDataToString(data) {
      let nestedArray = [], resultString;
      data.forEach((item) => nestedArray.push(item.name));
      resultString = nestedArray.join(', ');
      return resultString;
    },
    checkIfInFavorites(id){
      // Change to check in plex
      axios.get(`https://api.themoviedb.org/3/${this.tmdbType}/${id}/account_states?api_key=${storage.apiKey}&session_id=${storage.sessionId}`)
      .then(function(resp){
          this.favorite = resp.data.favorite;
          this.favoriteChecked = true;
          this.movieLoaded = true;
      }.bind(this))
    },
    // Toggle the downloading status if admin
    toggleFavorite(){
      let favoriteInvert = !this.favorite;
      this.favorite = '';
      axios.post(`https://api.themoviedb.org/3/account/${storage.userId}/favorite?api_key=${storage.apiKey}&session_id=${storage.sessionId}`, {
        'media_type': 'movie',
        'media_id': this.id,
        'favorite': favoriteInvert
      })
      .then(function(resp){
        this.favorite = favoriteInvert;
        eventHub.$emit('updateFavorite');
      }.bind(this));
    },
    // Send a request for a specific movie
    sendRequest(){
      this.requested = ''
      axios({
        method: 'post', //you can set what request you want to be
        url: `https://api.kevinmidboe.com/api/v1/plex/request/${this.id}?type=${this.mediaType}`,
        headers: {
          authorization: storage.token
        }
      })
      // axios.post(`https://api.kevinmidboe.com/api/v1/plex/request/${this.id}?type=${this.mediaType}`, {}, {
      //   authorization: storage.token
      // })
      // axios.post(`https://api.kevinmidboe.com/api/v1/plex/request/${this.id}?api_key=${storage.apiKey}&session_id=${storage.sessionId}`, {
      .then(function(resp){
        if (resp.data.success)
          this.requested = true;
        else
          this.requested = false;
      }.bind(this));
    },
    openTmdb(){
      window.location.replace('https://www.themoviedb.org/' + this.tmdbType + '/' + this.id)
    },
    // Search torrents by query
    // searchForTorrents() {
    //   axios.get(`https://apollo.kevinmidboe.com/api/v1/plex/request/${id}?type=${'movie'}&api_key=${storage.apiKey}&language=en-US`)
    // },
  },
  watch: {
    id: function(val){
      this.fetchMovie(val);
    }
  },
  created(){
    this.fetchMovie(this.id);
    console.log('admin: ', this.admin)
  }
}
</script>

<style lang="scss">
@import "./src/scss/variables";
@import "./src/scss/media-queries";

.movie{
  &__wrap{
    display: flex;
    &--page{
      max-width: 768px;
      position: relative;
      margin: 0 auto;
    }
    &--header{
      align-items: center;
      height: 100%;
    }
    &--main{
      display: flex;
      flex-wrap: wrap;
      flex-direction: column;
      @include tablet-min{
        flex-direction: row;
      }
    }
  }
  &__header{
    height: 250px;
    position: relative;
    background-size: cover;
    background-repeat: no-repeat;
    background-position: 50% 50%;
    background-color: $c-dark;
    @include tablet-min{
      height: 350px;
      &--page{
        height: 384px;
      }
    }
    &:before{
      content: "";
      display: block;
      position: absolute;
      top: 0;
      left: 0;
      z-index: 0;
      width: 100%;
      height: 100%;
      background: rgba($c-dark, 0.85);
    }
  }
    &__poster{
      display: none;
      @include tablet-min{
        background: $c-dark;
        display: block;
        position: absolute;
        width: calc(45% - 40px);
        top: 40px;
        left: 40px;
      }
    }
      &__img{
        display: block;
        width: 100%;
        opacity: 0;
        transform: scale(0.97) translateZ(0);
        transition: opacity 0.5s ease, transform 0.5s ease;
        &.is-loaded{
          opacity: 1;
          transform: scale(1);
        }
      }
    &__title{
      position: relative;
      padding: 20px;
      color: $c-green;
      text-align: center;
      width: 100%;
      @include tablet-min{
        width: 55%;
        text-align: left;
        margin-left: 45%;
        padding: 30px 30px 30px 40px;
      }
      &-text{
        font-weight: 500;
        line-height: 1.4;
        font-size: 24px;
        @include tablet-min{
          font-size: 30px;
        }
        span{
          display: block;
          font-size: 14px;
          font-weight: 300;
          color: rgba($c-white, 0.7);
          margin-top: 10px;
        }
      }
    }
  &__main{
    background: $c-light;
    min-height: calc(100vh - 250px);
    @include tablet-min{
      min-height: 0;
    }
  }
    &__actions{
      text-align: center;
      width: 100%;
      order: 2;
      padding: 20px;
      border-top: 1px solid rgba($c-dark, 0.05);
      @include tablet-min{
        order: 1;
        width: 45%;
        padding: 185px 0 40px 40px;
        border-top: 0;
      }
      &-link{
        display: flex;
        align-items: center;
        text-decoration: none;
        text-transform: uppercase;
        color: rgba($c-dark, 0.5);
        transition: color 0.5s ease;
        font-size: 11px;
        padding: 10px 0;
        border-bottom: 1px solid rgba($c-dark, 0.05);
        &:hover{
          color: rgba($c-dark, 0.75);
        }
        &.active{
          color: $c-dark;
        }
        &.pending{
          color: #f8bd2d;
        }
      }
      &-icon{
        width: 18px;
        height: 18px;
        margin: 0 10px 0 0;
        fill: rgba($c-dark, 0.5);
        transition: fill 0.5s ease, transform 0.5s ease;
        &.waiting{
          transform: scale(0.8, 0.8);
        }
        &.pending{
          fill: #f8bd2d;
        }
      }
      &-link:hover &-icon{
        fill: rgba($c-dark, 0.75);
        cursor: pointer;
      }
      &-link.active &-icon{
        fill: $c-green;
      }
      &-text{
        display: block;
        padding-top: 2px;
        cursor: pointer;
      }
    }
    &__info{
      width: 100%;
      padding: 20px;
      order: 1;
      @include tablet-min{
        order: 2;
        padding: 40px;
        width: 55%;
        margin-left: 45%;
      }
    }
    &__actions + &__info{
      margin-left: 0;
    }
      &__description{
        font-weight: 300;
        font-size: 13px;
        line-height: 1.8;
        margin-bottom: 20px;
        @include tablet-min{
          margin-bottom: 30px;
          font-size: 14px;
        }
      }
      &__details{
        &-block{
          float: left;
        }
        &-block:not(:last-child){
          margin-bottom: 20px;
          margin-right: 20px;
          @include tablet-min{
            margin-bottom: 30px;
            margin-right: 30px;
          }
        }
        &-title{
          margin: 0;
          font-weight: 400;
          text-transform: uppercase;
          font-size: 14px;
          color: $c-green;
          @include tablet-min{
            font-size: 16px;
          }
        }
        &-text{
          font-weight: 300;
          font-size: 14px;
          margin-top: 5px;
        }
      }
    &__admin{
      width: 100%;
      padding: 20px;
      order: 2;
      @include tablet-min{
        order: 3;
        padding: 40px;
        padding-top: 0px;
        width: 100%;
      }
      &-title{
          margin: 0;
          font-weight: 400;
          text-transform: uppercase;
          text-align: center;
          font-size: 14px;
          color: $c-green;
          padding-bottom: 20px;
          @include tablet-min{
            font-size: 16px;
          }
        }
    }
}
</style>
