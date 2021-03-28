<template>
    <div class="container mx-auto p-4">
        <div class="text-5xl mb-4">Better YouTube Algorithm</div>
        <div class="container mx-auto p-4">
            <button class="bg-gray-200 border-gray-400 py-2 px-4 rounded border-4 m-1 focus:outline-none"
                    @click="listChannels">
                List channels
            </button>
            <button class="bg-gray-200 border-gray-400 py-2 px-4 rounded border-4 m-1 focus:outline-none"
                    @click="listVideos">
                List videos
            </button>
        </div>
        <div class="container mx-auto p-4">
            <div v-for="subscriptionSnippet in mySubscriptionsSnippets">
                <div class="image-rounder" style="float:left; margin:0.5em;">
                    <img :src="subscriptionSnippet.thumbnails.default.url"
                         :title="subscriptionSnippet.title"
                         :alt="`Channel image for ${subscriptionSnippet.title}`"
                    />
                </div>
            </div>
        </div>
        <div style="clear:both;"></div>
        <div class="m-3">
            <p><b>Duration:</b> (minutes) min: <input v-model="minDurationFilter" /> max: <input v-model="maxDurationFilter" /></p>
        </div>
        <div class="container mx-auto p-4">
            <div v-for="videoItem in sortedVideos">
                <div class="m-3" :style="`float:left; max-width:${videoItem.snippet.thumbnails.medium.width}px`">
                    <a :href="`https://www.youtube.com/watch?v=${videoItem.id}`" target="_blank">
                        <img :src="videoItem.snippet.thumbnails.medium.url" />
                    </a>
                    <p><b>{{ videoItem.snippet.title }}</b></p>
                    <p>
                        <span :title="videoItem.snippet.publishedAt">{{ time_ago(videoItem.snippet.publishedAt) }}</span>
                        <span style="float:right;">{{ videoItem.snippet.channelTitle }}</span>
                    </p>
                    <p>
                        Duration: {{ durationToString(videoItem.contentDetails.duration) }}
                    </p>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
module.exports = {
    data: function() {
        return {
            mySubscriptionsSnippets: [],
            latestVideos: [],

            minDurationFilter: '0',
            maxDurationFilter: '100000000',
        }
    },
    computed: {
        filteredVideos: function() {
            return this.latestVideos.filter((video) => {
                const videoDuration = moment.duration(video.contentDetails.duration)
                const minDuration = moment.duration({ minutes: parseFloat(this.minDurationFilter) })
                const maxDuration = moment.duration({ minutes: parseFloat(this.maxDurationFilter) })
                return videoDuration > minDuration && videoDuration < maxDuration
            })
        },

        sortedVideos: function() {
            // FIXME: This sorts in-place - should probably clone the array first, but Vue docs imply to do this...!
            return this.filteredVideos.sort((a, b) => Date.parse(a.snippet.publishedAt) < Date.parse(b.snippet.publishedAt) ? 1 : -1)
        },
    },
    methods: {
        listChannels: function() {
            queryMyChannelId()
                .then((myChannelId) => {
                    return gapi.client.youtube.subscriptions.list({
                        'part': 'snippet,contentDetails',
                        'maxResults': '50', // FIXME: If more results than this, then need to handle pagination: https://developers.google.com/youtube/v3/guides/implementation/pagination
                        'channelId': myChannelId,
                    })
                })
                .then((response) => {
                    this.mySubscriptionsSnippets = response.result.items.map((item) => item.snippet)
                })
        },

        listVideos: function() {
            // For each channel, list its most recently published videos
            this.latestVideosSnippets = []
            for(const channelSnippet of this.mySubscriptionsSnippets) {
                gapi.client.youtube.channels.list({
                    'part': 'contentDetails',
                    'maxResults': '10', // FIXME: If more results than this, then need to handle pagination: https://developers.google.com/youtube/v3/guides/implementation/pagination
                    'id': channelSnippet.resourceId.channelId,
                })
                    .then((response) => {
                        const uploadsPlaylistId = response.result.items[0].contentDetails.relatedPlaylists.uploads
                        return gapi.client.youtube.playlistItems.list({
                            part: 'snippet',
                            maxResults: '10', // FIXME: If more results than this, then need to handle pagination: https://developers.google.com/youtube/v3/guides/implementation/pagination
                            playlistId: uploadsPlaylistId,
                        })
                    })
                    .then((response) => {
                        return response.result.items.map((item) => item.snippet)
                    })
                    .then((playlistItemSnippets) => {
                        // TODO: This batching isn't needed here, but we could use it everywhere
                        // TODO: Pull this out into a function batch(array, batch_size, function)
                        //         that calls function on batches of inputs from array
                        // TODO: Maybe:
                        // TODO:     Merge for loop above into a Promise.all() style merge of all channels
                        // TODO:     Call all the subsequent methods on the collection of all channels
                        const batch_size = 50
                        const requests = []
                        while(playlistItemSnippets.length) {
                            const ids_batch = playlistItemSnippets.splice(0, batch_size).map((snippet) => snippet.resourceId.videoId)
                            requests.push(gapi.client.youtube.videos.list({
                                part: 'snippet,contentDetails',
                                id: ids_batch,
                            }))
                        }
                        return Promise.all(requests)
                    })
                    .then((responseArray) => {
                        // TODO: Probably use reduce() to flatten the arrays
                        responseArray.map((response) => {
                            this.latestVideos.push(...response.result.items)
                        })
                    })
            }
        },

        /**
         * From https://stackoverflow.com/a/12475270/598749
         * @param time Either a Date object, a numeric timestamp or a date string
         * @returns string Past and previous time formats like '2 days ago' or '10 minutes from now'
         */
        time_ago: function(time) {

            switch (typeof time) {
                case 'number':
                    break;
                case 'string':
                    time = +new Date(time);
                    break;
                case 'object':
                    if (time.constructor === Date) time = time.getTime();
                    break;
                default:
                    time = +new Date();
            }
            var time_formats = [
                [60, 'seconds', 1], // 60
                [120, '1 minute ago', '1 minute from now'], // 60*2
                [3600, 'minutes', 60], // 60*60, 60
                [7200, '1 hour ago', '1 hour from now'], // 60*60*2
                [86400, 'hours', 3600], // 60*60*24, 60*60
                [172800, 'Yesterday', 'Tomorrow'], // 60*60*24*2
                [604800, 'days', 86400], // 60*60*24*7, 60*60*24
                [1209600, 'Last week', 'Next week'], // 60*60*24*7*4*2
                [2419200, 'weeks', 604800], // 60*60*24*7*4, 60*60*24*7
                [4838400, 'Last month', 'Next month'], // 60*60*24*7*4*2
                [29030400, 'months', 2419200], // 60*60*24*7*4*12, 60*60*24*7*4
                [58060800, 'Last year', 'Next year'], // 60*60*24*7*4*12*2
                [2903040000, 'years', 29030400], // 60*60*24*7*4*12*100, 60*60*24*7*4*12
                [5806080000, 'Last century', 'Next century'], // 60*60*24*7*4*12*100*2
                [58060800000, 'centuries', 2903040000] // 60*60*24*7*4*12*100*20, 60*60*24*7*4*12*100
            ];
            var seconds = (+new Date() - time) / 1000,
                token = 'ago',
                list_choice = 1;

            if (seconds == 0) {
                return 'Just now'
            }
            if (seconds < 0) {
                seconds = Math.abs(seconds);
                token = 'from now';
                list_choice = 2;
            }
            let i = 0, format;
            while (format = time_formats[i++])
                if (seconds < format[0]) {
                    if (typeof format[2] == 'string')
                        return format[list_choice];
                    else
                        return Math.floor(seconds / format[2]) + ' ' + format[1] + ' ' + token;
                }
            return time;
        },

        durationToString(duration) {
            let numMins = Math.trunc(moment.duration(duration).as('minutes'))
            if (numMins < 1) {
                numMins = '< 1'
            }
            return numMins + ' minutes'
        },
    },
    mounted: function() {
        gapi.load("client:auth2", function() {
            gapi.auth2.init({client_id: google_credentials.web.client_id});
            authenticate().then(loadClient)
        });
    },
}

function queryMyChannelId() {
    return gapi.client.youtube.channels.list({"part": ["snippet"], "mine": true})
        .then((response) => {
            return response.result.items[0].id;
        })
}

function authenticate() {
    return gapi.auth2.getAuthInstance()
        .signIn({scope: "https://www.googleapis.com/auth/youtube.readonly"})
        .then(function() { console.log("Sign-in successful"); },
            function(err) { console.error("Error signing in", err); });
}

function loadClient() {
    gapi.client.setApiKey(google_credentials.api_key);
    return gapi.client.load("https://www.googleapis.com/discovery/v1/apis/youtube/v3/rest")
        .then(function() { console.log("GAPI client loaded for API"); },
            function(err) { console.error("Error loading GAPI client for API", err); });
}

/**
 * Fisher-Yates Shuffle, from https://stackoverflow.com/a/2450976/598749
 * Shuffles an array in-place.
 */
function shuffle(array) {
  var currentIndex = array.length, temporaryValue, randomIndex;

  // While there remain elements to shuffle...
  while (0 !== currentIndex) {

    // Pick a remaining element...
    randomIndex = Math.floor(Math.random() * currentIndex);
    currentIndex -= 1;

    // And swap it with the current element.
    temporaryValue = array[currentIndex];
    array[currentIndex] = array[randomIndex];
    array[randomIndex] = temporaryValue;
  }

  return array;
}

/**
 * Randomly sample from an array without replacement, from https://stackoverflow.com/a/19270021/598749
 */
function getRandom(arr, n) {
    var result = new Array(n),
        len = arr.length,
        taken = new Array(len);
    if (n > len)
        throw new RangeError("getRandom: more elements taken than available");
    while (n--) {
        var x = Math.floor(Math.random() * len);
        result[n] = arr[x in taken ? taken[x] : x];
        taken[x] = --len in taken ? taken[len] : len;
    }
    return result;
}

</script>

<style>
    .image-rounder {
        width: 88px;
        height: 88px;
        position: relative;
        overflow: hidden;
        border-radius: 50%;
    }

    .image-rounder img {
        display: inline;
        margin: 0 auto;
        height: 100%;
        width: auto;
    }
</style>
