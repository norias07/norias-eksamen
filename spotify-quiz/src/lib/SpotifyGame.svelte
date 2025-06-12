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
    .game-container {
        background: rgba(0, 0, 0, 0.85);
        border-radius: 20px;
        padding: 40px;
        max-width: 600px;
        width: 90%;
        margin: 0 auto;
        text-align: center;
        box-shadow: 0 20px 40px rgba(0, 0, 0, 0.4);
        backdrop-filter: blur(15px);
        border: 1px solid rgba(29, 185, 84, 0.2);
        color: white;
    }

    h1 {
        font-size: 2.5em;
        font-weight: 800;
        margin-bottom: 30px;
        background: linear-gradient(45deg, #1db954, #1ed760);
        -webkit-background-clip: text;
        -webkit-text-fill-color: transparent;
        background-clip: text;
        letter-spacing: -1px;
    }

    .setup-section {
        margin-bottom: 30px;
    }

    .auth-section {
        background: rgba(29, 185, 84, 0.15);
        border: 2px solid #1db954;
        border-radius: 15px;
        padding: 25px;
        margin-bottom: 25px;
    }


    .auth-section h3 {
      color: #1ed760;
      margin-bottom: 10px;
      font-weight: 700;
    }
  
    .auth-section p {
      color: rgba(255, 255, 255, 0.8);
      margin-bottom: 20px;
      font-size: 14px;
    }
  
    .auth-button {
      background: linear-gradient(135deg, #1db954, #1ed760);
      color: #ffffff;
      border: none;
      padding: 15px 35px;
      border-radius: 50px;
      font-size: 16px;
      font-weight: 700;
      cursor: pointer;
      transition: all 0.3s cubic-bezier(0.25, 0.46, 0.45, 0.94);
      letter-spacing: 0.5px;
    }
  
    .auth-button:hover {
      background: linear-gradient(135deg, #1ed760, #21e065);
      transform: translateY(-3px) scale(1.02);
      box-shadow: 0 15px 30px rgba(29, 185, 84, 0.4);
    }
  
    .game-mode-selection h3 {
      color: #ffffff;
      margin-bottom: 15px;
      font-weight: 600;
    }
  
    .controls {
      display: flex;
      justify-content: center;
      gap: 15px;
      flex-wrap: wrap;
      margin: 20px 0;
    }
    .game-mode {
      background: rgba(255, 255, 255, 0.1);
      border: 2px solid rgba(29, 185, 84, 0.6);
      color: #ffffff;
      padding: 12px 25px;
      border-radius: 25px;
      cursor: pointer;
      transition: all 0.3s ease;
      font-weight: 600;
      margin: 0 8px;
    }
  
    .game-mode.active {
      background: linear-gradient(135deg, #1db954, #1ed760);
      border-color: #1ed760;
      box-shadow: 0 5px 15px rgba(29, 185, 84, 0.3);
    }
  
    .game-mode:hover {
      background: rgba(29, 185, 84, 0.2);
      transform: translateY(-2px);
    }
  
    .score {
      font-size: 1.3em;
      margin: 20px 0;
      color: #1ed760;
      font-weight: 700;
      background: rgba(29, 185, 84, 0.1);
      padding: 12px 20px;
      border-radius: 25px;
      border: 1px solid rgba(29, 185, 84, 0.3);
    }
  
    .song-info {
      background: linear-gradient(145deg, rgba(255, 255, 255, 0.08), rgba(255, 255, 255, 0.02));
      border: 1px solid rgba(255, 255, 255, 0.1);
      border-radius: 20px;
      padding: 25px;
      margin: 25px 0;
      min-height: 180px;
      display: flex;
      flex-direction: column;
      justify-content: center;
      backdrop-filter: blur(10px);
    }
  
    .album-cover {
      width: 120px;
      height: 120px;
      border-radius: 12px;
      margin: 0 auto 15px;
      object-fit: cover;
      box-shadow: 0 15px 35px rgba(0, 0, 0, 0.4);
      border: 2px solid rgba(29, 185, 84, 0.3);
    }
  
    .progress {
      width: 100%;
      height: 6px;
      background: rgba(255, 255, 255, 0.2);
      border-radius: 3px;
      margin: 15px 0;
      overflow: hidden;
      border: 1px solid rgba(255, 255, 255, 0.1);
    }
  
    .progress-bar {
      height: 100%;
      background: linear-gradient(90deg, #1db954, #1ed760);
      transition: width 0.1s linear;
      border-radius: 3px;
    }
  
    .play-button {
      background: linear-gradient(135deg, #1db954, #1ed760);
      color: #ffffff;
      border: none;
      padding: 15px 30px;
      border-radius: 50px;
      font-size: 16px;
      font-weight: 700;
      cursor: pointer;
      transition: all 0.3s cubic-bezier(0.25, 0.46, 0.45, 0.94);
      letter-spacing: 0.5px;
    }
  
    .play-button:hover {
      background: linear-gradient(135deg, #1ed760, #21e065);
      transform: translateY(-2px) scale(1.05);
      box-shadow: 0 12px 25px rgba(29, 185, 84, 0.4);
    }
  
    .play-button:disabled {
      background: linear-gradient(135deg, #666666, #777777);
      cursor: not-allowed;
      transform: none;
      box-shadow: none;
    }
  
    .skip-button {
      background: linear-gradient(135deg, #ff6b35, #ff8c42);
      color: #ffffff;
      border: none;
      padding: 15px 25px;
      border-radius: 50px;
      font-size: 16px;
      font-weight: 700;
      cursor: pointer;
      transition: all 0.3s ease;
    }
  
    .skip-button:hover {
      background: linear-gradient(135deg, #ff8c42, #ffa366);
      transform: translateY(-2px);
      box-shadow: 0 10px 20px rgba(255, 107, 53, 0.3);
    }
  
    .guess-section {
      margin: 25px 0;
      background: rgba(255, 255, 255, 0.05);
      padding: 20px;
      border-radius: 15px;
      border: 1px solid rgba(255, 255, 255, 0.1);
    }
  
    .guess-input {
      background: rgba(25, 20, 20, 0.6);
      border: 2px solid rgba(29, 185, 84, 0.6);
      color: #ffffff;
      padding: 15px 20px;
      border-radius: 12px;
      font-size: 16px;
      width: 100%;
      margin-bottom: 15px;
      transition: all 0.3s ease;
      font-family: inherit;
    }
  
    .guess-input:focus {
      outline: none;
      border-color: #1ed760;
      box-shadow: 0 0 0 3px rgba(29, 185, 84, 0.2);
      background: rgba(25, 20, 20, 0.8);
    }
  
    .guess-input::placeholder {
      color: rgba(255, 255, 255, 0.5);
    }
  
    .guess-button {
      background: linear-gradient(135deg, #1db954, #1ed760);
      color: #ffffff;
      border: none;
      padding: 12px 25px;
      border-radius: 25px;
      font-size: 16px;
      font-weight: 600;
      cursor: pointer;
      margin: 8px;
      transition: all 0.3s ease;
    }
  
    .guess-button:hover {
      background: linear-gradient(135deg, #1ed760, #21e065);
      transform: translateY(-1px);
    }
  
    .next-button {
      background: linear-gradient(135deg, #ff6b35, #ff8c42);
      color: #ffffff;
      border: none;
      padding: 12px 25px;
      border-radius: 25px;
      font-size: 16px;
      font-weight: 600;
      cursor: pointer;
      margin: 8px;
      transition: all 0.3s ease;
    }
  
    .next-button:hover {
      background: linear-gradient(135deg, #ff8c42, #ffa366);
      transform: translateY(-1px);
    }
  
    .result {
      margin: 20px 0;
      padding: 18px 25px;
      border-radius: 12px;
      font-weight: 700;
      font-size: 16px;
      letter-spacing: 0.3px;
    }
  
    .result.correct {
      background: linear-gradient(135deg, rgba(29, 185, 84, 0.2), rgba(30, 215, 96, 0.15));
      border: 2px solid #1db954;
      color: #1ed760;
    }
  
    .result.incorrect {
      background: linear-gradient(135deg, rgba(255, 0, 0, 0.15), rgba(255, 69, 69, 0.1));
      border: 2px solid #ff0000;
      color: #ff6b6b;
    }
  
    .loading {
      color: #1ed760;
      font-size: 18px;
      margin: 25px 0;
      font-weight: 600;
      animation: pulse 1.8s infinite ease-in-out;
    }
  
    @keyframes pulse {
      0%, 100% { 
        opacity: 1; 
        transform: scale(1);
      }
      50% { 
        opacity: 0.6; 
        transform: scale(1.02);
      }
    }
  
    @media (max-width: 600px) {
      .game-container {
        padding: 25px 20px;
        margin: 20px;
      }
      
      h1 {
        font-size: 2em;
        margin-bottom: 25px;
      }
      
      .controls {
        flex-direction: column;
        align-items: center;
      }
  
      .game-mode {
        margin: 5px 0;
        width: 200px;
      }
  
      .auth-button, .play-button, .skip-button {
        width: 100%;
        max-width: 280px;
      }
    }
</style>