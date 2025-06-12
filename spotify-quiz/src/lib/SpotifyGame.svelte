<script>
import { onMount } from 'svelte';
import { browser } from '$app/environment';
import { goto } from '$app/navigation'; 

const CLIENT_ID = '2f447864907247b0b6cf278270612262'; 
let REDIRECT_URI = 'http://localhost:5173/callback'; 
const AUTH_ENDPOINT = 'https://accounts.spotify.com/authorize';
const RESPONSE_TYPE = 'token';
const SCOPES = [
    'user-read-private',
    'user-read-email',
    'playlist-read-private',
    'playlist-modify-public',
    'playlist-modify-private'
].join('%20');

let accessToken = null;
let currentTrack= null;
let gameMode = 'artist';
let score = 0;
let round = 1;
let maxRounds = 10;
let audioPlayer = null
let isLoading = false;
let isPlaying = false;
let showGuessSection = false;
let showResult = false;
let resultMessage = '';
let resultClass = '';
let progressWidth = 0;
let guessInput = '';
let gameStarted = false;

onMount(() => {
    if (browser) {
        const REDIRECT_URI = 'https://norias-eksamen.vercel.app/';

        const urlParams = new URLSearchParams(window.location.hash.substring(1));
        const token = urlParams.get('access_token');
        if (token) {
            accessToken = token;
            gameStarted = true;
            loadNewSong();

            history.replaceState(null, '', window.location.pathname);
        }
    }
});
function authenticateSpotify() {
      if (!browser) return;
      
      const authUrl = `https://accounts.spotify.com/authorizeclient_id=2f447864907247b0b6cf278270612262&response_type=token&redirect_uri=https://norias-eksamen.vercel.app/&scope=user-read-private%20user-read-email`.replace(/\n/g, '') +
          `client_id=${CLIENT_ID}&` +
          `response_type=token&` +
          `redirect_uri=${encodeURIComponent(REDIRECT_URI)}&` +
          `scope=${encodeURIComponent(SCOPES)}`;
      
          console.log('Redirecting to Spotify authentication...');
      window.location.href = authUrl;
 
    }
    function setGameMode(mode) {
      gameMode = mode;
    }
  
    async function loadNewSong() {
      isLoading = true;
      showGuessSection = false;
      showResult = false;
      isPlaying = false;
      progressWidth = 0;
  
      try {
        const genres = ['pop', 'rock', 'hip-hop', 'electronic', 'indie', 'alternative'];
        const randomGenre = genres[Math.floor(Math.random() * genres.length)];
        
        const response = await fetch(`https://api.spotify.com/v1/search?q=genre:${randomGenre}&type=track&limit=50&market=NO`, {
          headers: {
            'Authorization': `Bearer ${accessToken}`
          }
        });
  
        if (!response.ok) {
          throw new Error('Kunne ikke hente sanger fra Spotify');
        }
  
        const data = await response.json();
        const tracks = data.tracks.items.filter(track => track.preview_url);
        
        if (tracks.length === 0) {
          throw new Error('Ingen sanger med preview tilgjengelig');
        }
  
        currentTrack = tracks[Math.floor(Math.random() * tracks.length)];
        
      } catch (error) {
        console.error('Feil ved lasting av sang:', error);
        loadDemoSong();
      }
      
      isLoading = false;
    }
  
    function loadDemoSong() {
      currentTrack = {
        name: "Shape of You",
        artists: [{name: "Ed Sheeran"}],
        album: {
          images: [{url: "https://via.placeholder.com/300x300/1db954/ffffff?text=🎵"}]
        },
        preview_url: null
      };
    }
  
    function playPreview() {
      if (!currentTrack) return;
  
      if (currentTrack.preview_url && !isPlaying) {
        if (audioPlayer) {
          audioPlayer.pause();
        }
        
        audioPlayer = new Audio(currentTrack.preview_url);
        audioPlayer.play();
        isPlaying = true;
        
        let progress = 0;
        const progressInterval = setInterval(() => {
          progress += 1;
          progressWidth = (progress / 30) * 100;
          
          if (progress >= 30) {
            clearInterval(progressInterval);
            stopPreview();
          }
        }, 1000);
        
        setTimeout(() => {
          showGuessSection = true;
        }, 5000);
        
      } else if (isPlaying) {
        stopPreview();
      } else {
        // Demo mode
        isPlaying = true;
        showGuessSection = true;
        
        setTimeout(() => {
          isPlaying = false;
        }, 3000);
      }
    }
  
    function stopPreview() {
      if (audioPlayer) {
        audioPlayer.pause();
        audioPlayer = null;
      }
      isPlaying = false;
      progressWidth = 0;
    }
  
    function submitGuess() {
      const guess = guessInput.trim().toLowerCase();
      if (!guess) return;
  
      let correct = false;
      let correctAnswer = '';
  
      if (gameMode === 'artist') {
        correctAnswer = currentTrack.artists[0].name;
        correct = currentTrack.artists.some(artist => 
          artist.name.toLowerCase().includes(guess) || 
          guess.includes(artist.name.toLowerCase())
        );
      } else {
        correctAnswer = currentTrack.name;
        correct = currentTrack.name.toLowerCase().includes(guess) || 
                 guess.includes(currentTrack.name.toLowerCase());
      }
  
      if (correct) {
        score += 10;
        resultClass = 'correct';
        resultMessage = `🎉 Riktig! Det var "${correctAnswer}"`;
      } else {
        resultClass = 'incorrect';
        resultMessage = `❌ Feil. Det riktige svaret var "${correctAnswer}"`;
      }
  
      showResult = true;
      guessInput = '';
      
      if (round >= maxRounds) {
        setTimeout(() => {
          alert(`Spillet er ferdig! Du fikk ${score} av ${maxRounds * 10} mulige poeng.`);
          resetGame();
        }, 2000);
      }
    }
  
    function nextSong() {
      stopPreview();
      round++;
      loadNewSong();
    }
  
    function skipSong() {
      stopPreview();
      loadNewSong();
    }
  
    function resetGame() {
      score = 0;
      round = 1;
      loadNewSong();
    }
  
    function handleKeyPress(event) {
      if (event.key === 'Enter' && showGuessSection) {
        submitGuess();
      }
    }

</script>

<svelte:window on:keypress={handleKeyPress} />
  
<div class="game-container">
  <h1>🎵 Spotify Musikkgjettelek</h1>
  
  {#if !gameStarted}
    <div class="setup-section"></div>
      <div class="auth-section">
        <h3>Koble til Spotify</h3>
        <p>For å spille trenger du å koble til Spotify-kontoen din</p>
        <button class="auth-button" on:click={authenticateSpotify}>
          Koble til Spotify
        </button>
      
      <div class="game-mode-selection">
        <h3>Velg spillmodus:</h3>
        <div class="controls">
          <button 
            class="game-mode {gameMode === 'artist' ? 'active' : ''}" 
            on:click={() => setGameMode('artist')}
          >
            Gjett Artist
          </button>
          <button 
            class="game-mode {gameMode === 'song' ? 'active' : ''}" 
            on:click={() => setGameMode('song')}
          >
            Gjett Sang
          </button>
        </div>
      </div>
    </div>
  {:else}
    <div class="game-section">
      <div class="score">
        Poeng: <span>{score}</span> | Runde: <span>{round}</span>/{maxRounds}
      </div>
      
      <div class="song-info">
        {#if isLoading}
          <div class="loading">Laster ny sang...</div>
        {:else if currentTrack}
          <img 
            class="album-cover" 
            src={currentTrack.album.images[0]?.url || ''} 
            alt="Album cover"
          >
          {#if isPlaying && currentTrack.preview_url}
            <div class="progress">
              <div class="progress-bar" style="width: {progressWidth}%"></div>
            </div>
          {/if}
        {/if}
      </div>

      <div class="controls">
        <button 
          class="play-button" 
          on:click={playPreview}
          disabled={isLoading || !currentTrack}
        >
          {isPlaying ? '⏸️ Stopp' : '▶️ Spill Preview'}
        </button>
        <button class="skip-button" on:click={skipSong}>
          ⏭️ Bytt Sang
        </button>
      </div>

      {#if showGuessSection}
        <div class="guess-section">
          <input 
            type="text" 
            class="guess-input" 
            bind:value={guessInput}
            placeholder="Skriv ditt svar her..."
          >
          <div class="controls">
            <button class="guess-button" on:click={submitGuess}>Send Svar</button>
            <button class="next-button" on:click={nextSong}>Neste Sang</button>
          </div>
        </div>
      {/if}

      {#if showResult}
        <div class="result {resultClass}">
          {resultMessage}
        </div>
      {/if}
    </div>
  {/if}
</div>


<style>


</style>