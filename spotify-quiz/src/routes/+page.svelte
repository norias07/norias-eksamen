<script>
    import SpotifyGame from "$lib/SpotifyGame.svelte";
</script>

<svelte:head>
    <title>Spotify Quiz</title>
    <meta name="description" content="Test your Spotify knowledge with this fun quiz!">
    <link rel="preconnct" href="https://fonts.googleapis.com">
    <link rel="preconnct" href="https://fonts.gstatic.com" >
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet" >
</svelte:head>
<main>
    <SpotifyGame />
</main>
<style>
    :global(html) {
        font-family: 'inter', 'circular', 'helvetica neue', 'arial', 'sans-serif';
        background-color: #121212;
    }
    :global(body) {
    margin: 0;
    padding: 0;
    background-color: linear-gradient(135deg, #1db954, #191414);
    min-height: 100vh;
    display: flex;
    justify-content: center;
    align-items: center;
    }
    main {
        width: 100%;
        display: flex;
        justify-content: center;
        align-items: center;
        min-height: 100vh;
        padding: 20px;
    }
</style>

