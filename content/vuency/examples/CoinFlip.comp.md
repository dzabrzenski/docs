---
title: Coin Flip
order: 1
---

<div class="showcase">
  @[tasks/examples/CoinFlip]
</div>

### Script

```js
export default {
  data: () => ({
    answer: ''
  }),

  tasks (t, { timeout }) {
    return {
      flipCoin: t(function * () {
        // Take some time before giving answer.
        this.answer = yield timeout(1000)
          .then(() => Math.random() < 0.5 ? 'Heads' : 'Tails')
      })
      // Don't allow repeat calls to fire (one flip at a time!).
      .flow('drop')
       // Cleanup after each flip.
      .beforeStart(() => {
        this.answer = ''
      })
    }
  }
}
```

### Template

```html
<div class="coin-flip">
  <p> Heads or Tails? {{ answer }} </p>
  <button @click="flipCoin.run()">
    {{ flipCoin.isActive ? 'May the odds be with you...' : 'Flip coin'}}
  </button>
</div>
```
