<script lang="ts">
  import FlappyBird from './FlappyBird.svelte';
</script>

<svelte:head>
  <title>Flappy Bird Game</title>
  <meta name="description" content="A minimalistic Flappy Bird clone built in Svelte" />
</svelte:head>

<div class="flappy-outer">
  <FlappyBird />
</div>

<style>
.flappy-outer {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  min-height: 88vh;
  width: 100vw;
  margin: 0;
  background: none;
}
</style>
