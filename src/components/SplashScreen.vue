<template>
  <div class="splash-screen">
    <h1>Padira.</h1>
    <p>Loading...</p>
  </div>
</template>

<script>
export default {
  name: 'SplashScreen'
}
</script>

<style scoped>
.splash-screen {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: var(--white, #ffffff);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 9999;
  transition: opacity 0.5s ease;
}

.splash-screen h1 {
  font-size: 3rem;
  font-weight: 700;
  margin-bottom: 1rem;
}

.splash-screen p {
  font-size: 1.2rem;
  color: var(--gray-500, #737373);
}
</style>