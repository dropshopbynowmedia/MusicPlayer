<script>
import Playlist from './Playlist.svelte';
import TrackDetails from './TrackDetails.svelte';
import PLaybackControls from './PlaybackControls.svelte';
import Settings from './Settings.svelte';


const ipc = require('electron').ipcRenderer;
const fs = require('fs')
const path = require('path')
const storage = require('electron-json-storage');
const mm = require('music-metadata');

const dataPath = storage.getDataPath();


let trackName = "";
let trackArtist = "";
let trackAlbum = "";
let songList = null;
let songPlaying = false;
let playListVisible = false;
let shuffle = false;
let mute = false;
let loading = false;
let theme = "dark";

let duration = "00:00"
let timer = "00:00"

let files = null
let player = null

let search = ""

let slider = 100
let offsetWidth;

storage.has('path', function (error, hasKey) {
	if (error) throw error;
	if (hasKey) {
	storage.get('path', function (error, data) {
		if (error) throw error;
		scanDir([data.path.toString()]);
	});
	}
})

function setTheme(data) {
	var icons = document.body.querySelectorAll("svg");
	if (data.theme == "light") {
		theme = 'light'
		document.body.style.backgroundColor = "#F5F5F5"
		document.body.style.color = "#212529"

		icons.forEach(icon => {
			icon.style.color = "#212529";
		});
	}
	else if (data.theme == "dark") {
		theme = 'dark'
		document.body.style.backgroundColor = "#212121"
		document.body.style.color = "azure"

		icons.forEach(icon => {
			icon.style.color = "azure";
		});
	}
	else if (data.theme == "disco") {
		theme = 'disco'
		icons.forEach(icon => {
			icon.style.color = "azure";
		});
		
	}
}

storage.has('theme', function (error, hasKey) {
	if (error) throw error;
	if (hasKey) {
	storage.get('theme', function (error, data) {
		if (error) throw error;
		setTheme(data)
	});
	}
})


var walkSync = function (dir, filelist) {
	files = fs.readdirSync(dir);
	filelist = filelist || [];
	files.forEach(function (file) {
	if (fs.statSync(path.join(dir, file)).isDirectory()) {
		filelist = walkSync(path.join(dir, file), filelist);
	}
	else {
		if (file.endsWith('.mp3') || file.endsWith('.m4a')
		|| file.endsWith('.webm') || file.endsWith('.wav')
		|| file.endsWith('.aac') || file.endsWith('.ogg')
		|| file.endsWith('.opus')) {
		filelist.push(path.join(dir, file));
		}
	}
	});
	return filelist;
};

async function parseFiles(audioFiles) {
	var titles = []

	loading = true;

	for (const audioFile of audioFiles) {

	// await will ensure the metadata parsing is completed before we move on to the next file
	const metadata = await mm.parseFile(audioFile, { skipCovers: true });
	var data = {}
	var title = metadata.common.title
	var artist = metadata.common.artist
	if(title)
		data.title = metadata.common.title;
	else
		data.title = audioFile.split(path.sep).slice(-1)[0];
	if (artist)
		data.artist = metadata.common.artist;
	else
		data.artist = '';
	
	titles.push(data)
	
	}
	loading = false;

	return titles
}

async function scanDir(filePath) {
	if (!filePath || filePath[0] == 'undefined') return;

	var arr = walkSync(filePath[0]);
	var arg = {};
	var names = await parseFiles(arr)
	
	arg.files = arr;
	arg.path = filePath;
	arg.names = names

	startPlayer(arg)
}
    
function themeChange(data) {
	
	// console.log(data);
	setTheme(data)
	
}
ipc.on('theme-change', function (event, arg) {
	themeChange(arg)
});

ipc.on('selected-files', function (event, arg) {
	// console.log(arg);
	
	scanDir(arg)

});

function startPlayer(arg) {

	if (songPlaying) {
		player.pause();
		songPlaying = false;
	}
	songList = arg;
	// console.log(songList)
	var songArr = [];

	for (let i = 0; i < songList.files.length; i++) {
	songArr.push({
		title: songList.files[i],
		file: songList.files[i],
		name: songList.names[i].title,
		artist: songList.names[i].artist,
		howl: null,
		index: i
	});
	}

	storage.has('last-played', function (error, hasKey) {
		if (error) throw error;
		if (hasKey) {
			storage.get('last-played', function (error, data) {
			if (error) throw error;
			var index = arg.files.indexOf(data.path)
			
			if (index != -1){
				player = new Player(songArr, index);
			} else {
				player = new Player(songArr, 0);
			}

			playMusic()
			playMusic()
			});
		}
		else {
			player = new Player(songArr, 0);
			
			playMusic()
			playMusic()
		}
	})


}

function getTags(audioFile) {
	var titles = []
	const metadata = mm.parseFile(audioFile, { skipCovers: false })
	.then(metadata => {
		// console.log(metadata.common);
		var title = metadata.common.title
		var artist = metadata.common.artist
		var album = metadata.common.album

		if(title)
			trackName = title;
		else
			trackName = audioFile.split(path.sep).slice(-1)[0]
		if(artist)
			trackArtist = artist;
		else
			trackArtist = ""
		if (album)
			trackAlbum = album;
		else
			trackAlbum = ""
		var img = document.getElementById('picture')

		if (metadata.common.picture) {
			var picture = metadata.common.picture[0]
			img.style.display = "block";
			img.src = `data:${picture.format};base64,${picture.data.toString('base64')}`;
			img.addEventListener('load', function () {
				if (theme == 'disco') {
					var vibrant = new Vibrant(img, 128, 3);
					var swatches = vibrant.swatches()
					if (swatches['DarkMuted'])
						document.body.style.backgroundColor = swatches['DarkMuted'].getHex()
					else
						document.body.style.backgroundColor = "#212121"
					if (swatches['LightVibrant'])
						document.body.style.color = swatches['LightVibrant'].getHex()
					else 
						document.body.style.color = "azure"
				}
			})
		} else {
			img.style.display = "none";
		}
	})
	.catch(err => {
		console.error(err.message);
	});

	return titles
}


var seekToTime = function (event) {
	player.seek(event.offsetX / offsetWidth);
}
var playPlaylistSong = function (index) {
	player.skipTo(index);
}
var nextSong = function () {
	if (shuffle) {
		player.skip('random-next');
	}
	else {
		player.skip('next');
	}
	songPlaying = true;
}
var prevSong = function () {
	if (shuffle) {
		player.skip('random-prev');
	}
	else {
		player.skip('prev');
	}
	songPlaying = true;
}

var showPlaylist = function () {
	if (playListVisible) {
		playListVisible = false;
	// console.log(playListVisible)
	}
	else {
		playListVisible = true;
	// console.log(playListVisible)
	}
}

var playMusic = function () {
	
	if (songPlaying) {
		player.pause();
		songPlaying = false;
	}
	else {
		player.play();
		songPlaying = true;
	}

}

var toggleShuffle = function () {
	
	if (shuffle) {
		shuffle = false;
	}
	else {
		shuffle = true;
	}
}

var togglecheckbox = function() {
	if (mute) {
		mute = false;
		player.volume(slider / 100);
	}
	else {
		mute = true;
		player.volume(0);
	}
}

function randomize(array) {
    for (let i = array.length - 1; i > 0; i--) {
        let j = Math.floor(Math.random() * (i + 1));

        [array[i], array[j]] = [array[j], array[i]];
    }
    return array;
}


var Player = function (playlist, index) {
	this.playlist = playlist;
	this.index = index;
	this.randomIndex = index;
	this.randomArray = randomize(Array.from({length: playlist.length}, (_, i) => i))
}

    Player.prototype = {

      play: function (index) {
        var self = this;
        var sound;

        index = typeof index === 'number' ? index : self.index;
        var data = self.playlist[index];

        if (data.howl) {
          sound = data.howl;
        } else {
          sound = data.howl = new Howl({
            src: [data.file],
            html5: true,
            onplay: function () {
              duration = self.formatTime(Math.round(sound.duration()));
              requestAnimationFrame(self.step.bind(self));
            },
            onend: function () {
              if (shuffle) {
                self.skip('random');
              }
              else {
                self.skip('right');
              }
            }
          });
        }
        
        storage.set('last-played', { path: data.file }, function (error) {
          if (error) throw error
        })
        sound.play();
        getTags(data.file)

        self.index = index;
      },

      pause: function () {
        var self = this;

        var sound = self.playlist[self.index].howl;

        sound.pause();
      },

      skip: function (direction) {
        var self = this;

        var index = 0;
        if (direction === 'prev') {
          index = self.index - 1;
          if (index < 0) {
            index = self.playlist.length - 1;
          }
        }
        else if (direction === 'random-next') {
			self.randomIndex += 1
			if (self.randomIndex >= self.randomArray.length) {
				self.randomIndex = 0;
			}
			index = self.randomArray[self.randomIndex]
		}
		else if (direction === 'random-prev') {
			self.randomIndex -= 1
			if (self.randomIndex < 0) {
				self.randomIndex = self.randomArray.length - 1;
			}
			index = self.randomArray[self.randomIndex]
        }
        else {
          index = self.index + 1;
          if (index >= self.playlist.length) {
            index = 0;
          }
		}
		console.log(index);
		

        // var data = self.playlist[self.index];

        self.skipTo(index);
      },

      skipTo: function (index) {
        var self = this;

        if (self.playlist[self.index].howl) {
          self.playlist[self.index].howl.stop();
        }
        var data = self.playlist[index];

        if (!songPlaying) {
          songPlaying = true;
          self.play(index);
        }
        else
          self.play(index);

      },

      step: function () {
        var self = this;

        var sound = self.playlist[self.index].howl;

        var seek = sound.seek() || 0;
        timer = self.formatTime(Math.round(seek));
        progress.style.width = (((seek / sound.duration()) * 100) || 0) + '%';

        if (sound.playing()) {
          requestAnimationFrame(self.step.bind(self));
        }
      },
      formatTime: function (secs) {
        var minutes = Math.floor(secs / 60) || 0;
        var seconds = (secs - minutes * 60) || 0;

        return minutes + ':' + (seconds < 10 ? '0' : '') + seconds;
      },
      volume: function (val) {
        var self = this;
        Howler.volume(val);

      },
      seek: function (time) {
        var self = this;

        var sound = self.playlist[self.index].howl;

        if (sound.playing() || true) {
          sound.seek(sound.duration() * time);
          requestAnimationFrame(self.step.bind(self));
        }
      }
    }


$: if(player) {
	player.volume(slider/100)
	mute = false;
}

</script>

<div class="container-fluid">

	<div class="row">
	{#if playListVisible}
		<Playlist player={player} on:changeSong={(event) => playPlaylistSong(event.detail.index) }></Playlist>
	{/if}
		<div class="col-5  my-auto">
			{#if loading}
				<div class="spinner-border text-danger centerBlock" style="width: 5rem; height: 5rem;" role="status">
					<span class="sr-only">Loading...</span>
				</div>
			{:else}
				<img id="picture" alt="">
			{/if}
		</div>

		<div class="col">
			<div class="row">
			
				<div class="col-md-12 text-center">
					<TrackDetails trackName={trackName} trackArtist={trackArtist} 
					trackAlbum={trackAlbum} theme={theme}></TrackDetails>
				</div>
			
				<div class="col-md-12 text-center">
					<PLaybackControls on:prevSong={prevSong} on:nextSong={nextSong} 
					on:playMusic={playMusic} songPlaying={songPlaying} ></PLaybackControls>
				</div>
			
				<div class="col-md-12 text-center">
					<div id="timer"> {timer} </div>
					<div id="duration"> {duration} </div><br>
			
					<div class="progress" id="seek" bind:clientWidth={offsetWidth} on:click={e => seekToTime(e)}>
						<div class="progress-bar bg-danger" 
						style="transition: width .2s cubic-bezier(0.22, 0.61, 0.36, 1);"
							role="progressbar" id="progress" aria-valuemin="0" aria-valuemax="100"></div>
					</div>
				</div>
				<br>
				<div class="col-md-12" id="outerCtrl">
					<Settings on:showPlaylist={showPlaylist} on:toggleShuffle={toggleShuffle} shuffle={shuffle}
					on:togglecheckbox={togglecheckbox} bind:slider={slider} mute={mute}  ></Settings>
				</div>
			</div>
		</div>
	</div>
</div>