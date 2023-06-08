<script lang="ts">
  let quotes: string;
  let result: string;
  let similarityPercentage: number = 50;

  let decisionModalShown: boolean = false;
  let q0: string;
  let q1: string;
  let simString: string;
  let currentDecision: number = 0;
  let totalDecisions: number = 0;

  const openModal = (): Promise<void> => {
    return new Promise((resolve) => {
      decisionModalShown = true;
      setInterval(() => {
        const modal = document.getElementById('modal');
        if (modal) resolve();
      });
    });
  };

  const getDecision = (quote0: string, quote1: string): Promise<0 | 1 | 2> => {
    return new Promise(async (resolve) => {
      await openModal();
      q0 = quote0;
      q1 = quote1;

      const button0: Element = document.getElementById('0-button');
      const button1: Element = document.getElementById('1-button');
      const button2: Element = document.getElementById('2-button');

      button0.addEventListener('click', () => {
        decisionModalShown = false;
        button0.removeEventListener('click', () => {});
        resolve(0);
      });

      button1.addEventListener('click', () => {
        decisionModalShown = false;
        button1.removeEventListener('click', () => {});
        resolve(1);
      });

      button2.addEventListener('click', () => {
        decisionModalShown = false;
        button2.removeEventListener('click', () => {});
        resolve(2);
      });
    });
  };

  // similarity and editDistance stolen from https://stackoverflow.com/a/36566052/114157
  const editDistance = (s1: string, s2: string): number => {
    s1 = s1.toLowerCase();
    s2 = s2.toLowerCase();

    let costs = new Array();
    for (let i = 0; i <= s1.length; i++) {
      let lastValue = i;
      for (let j = 0; j <= s2.length; j++) {
        if (i == 0) costs[j] = j;
        else {
          if (j > 0) {
            let newValue = costs[j - 1];
            if (s1.charAt(i - 1) != s2.charAt(j - 1))
              newValue = Math.min(Math.min(newValue, lastValue), costs[j]) + 1;
            costs[j - 1] = lastValue;
            lastValue = newValue;
          }
        }
      }
      if (i > 0) costs[s2.length] = lastValue;
    }

    return costs[s2.length];
  };

  const similarity = (s1: string, s2: string): number => {
    s1 = s1.substring(1, s1.length - 1);
    s2 = s2.substring(1, s2.length - 1);

    let longer = s1;
    let shorter = s2;
    if (s1.length < s2.length) {
      longer = s2;
      shorter = s1;
    }
    let longerLength = longer.length;
    return (longerLength - editDistance(longer, shorter)) / longerLength;
  };

  const tokeniseQuotes = (quotes: string): string[] => {
    quotes = quotes.split('\n').join('');
    const tokens = quotes.match(
      /"([^"]*)"|'([^']*)'|`([^`]*)`|“([^“”]*)”|‘([^‘’]*)’|«([^«»]*)»|‹([^‹›]*)›|„([^„“]*)“/g
    );
    return tokens;
  };

  const similarityChange = (e) => {
    similarityPercentage = Math.max(0, Math.min(100, Number(e.target.value)));
    e.target.value = similarityPercentage;
  };

  const submit = async (e: SubmitEvent): Promise<void> => {
    let tokens = tokeniseQuotes(quotes);
    if (!tokens) {
      result = '';
      return;
    }
    const uniqueTokens: string[] = [];

    while (true) {
      const token = tokens[0];

      if (!token) break;
      tokens = tokens.splice(1);
      const indexesToRemove: number[] = [];
      let shouldPush = true;
      currentDecision = 0;
      totalDecisions = tokens.length;
      for (let i = 0; i < tokens.length; i++) {
        currentDecision = i;
        const otherToken = tokens[i];
        const sim = similarity(token, otherToken);

        simString = (sim * 100).toFixed(2);

        if (sim === 1) {
          indexesToRemove.push(i);
          continue;
        } else if (sim * 100 >= similarityPercentage) {
          const decision = await getDecision(token, otherToken);
          if (decision === 0) {
            // Keep first
            indexesToRemove.push(i);
            continue;
          } else if (decision === 1) {
            // Keep second
            uniqueTokens.push(otherToken);
            indexesToRemove.push(i);
            shouldPush = false;
            break;
          } else if (decision === 2) {
            // Keep both
            uniqueTokens.push(otherToken);
            indexesToRemove.push(i);
          }
        }
      }

      tokens = tokens.filter((_, i) => !indexesToRemove.includes(i));

      console.log(tokens);

      if (shouldPush) uniqueTokens.push(token);
    }

    result = uniqueTokens.join('\n');
    if (uniqueTokens.length === 0) {
      result = '';
    }

    const resultTextArea = document.getElementById('result');
    const nLines = uniqueTokens.length;

    resultTextArea.style.height = `${Math.max(nLines * 21, 50)}px`;
  };

  const textAreaChange = (e): void => {
    const nLines = e.target.value.split('\n').length;

    e.target.style.height = `${Math.max(nLines * 21, 50)}px`;
  };
</script>

<h1>Quote Duplicate Eliminator</h1>

<main>
  <div>
    <form on:submit|preventDefault={submit}>
      <p>
        Input quotes below. Make sure that quotes are surrounded by speech marks
        and are on separate lines.
        <br />
        It will match any of the following quotation mark types:
      </p>
      <pre>"quote" 'quote' `quote` “quote” ‘quote’ «quote» ‹quote› „quote“</pre>
      <p>
        Similarity percentage: <input
          style="display: inline-block; width: 50px;"
          type="number"
          on:change={similarityChange}
          value={similarityPercentage}
          min="0"
          max="100"
        />%
      </p>
      <p>
        If a comparison of two quotes is more than, or equal to {similarityPercentage}
        per cent similar, you will be asked what to do.
      </p>
      <textarea
        bind:value={quotes}
        on:input={textAreaChange}
        placeholder="Type quotes here..."
      />

      <br />
      <br />
      <input type="submit" value="Remove duplicates" />
    </form>
  </div>
  <div>
    <p>Removed duplicates:</p>
    <textarea
      bind:value={result}
      readonly
      id="result"
      placeholder="Results will generate here..."
    />
  </div>
</main>
<p style="position: fixed; left: 0; bottom: 0; padding: 0; margin: 0;">
  Hi Sam :)
</p>
{#if decisionModalShown}
  <div class="decision-modal-bg">
    <div class="decision-modal" id="modal">
      <div class="decision-progress">
        <div class="decision-current">{currentDecision + 1}</div>
        <div class="decision-slash">/</div>
        <div class="decision-total">{totalDecisions}</div>
      </div>
      <h2>
        These quotes look similar. <br />What would you like to do?
        <br />
        (Similarity: {simString}%)
      </h2>
      <p>
        <b>Quote 1:</b>
        {q0}
        <br />
        <b>Quote 2:</b>
        {q1}
      </p>
      <button id="0-button">Keep first</button>
      <button id="1-button">Keep second</button>
      <button id="2-button">Keep both</button>
    </div>
  </div>
{/if}

<style>
  .decision-progress {
    position: absolute;
    top: 5px;
    right: 5px;
    width: 40px;
    height: 40px;
  }

  .decision-current {
    position: absolute;
    top: 0;
    left: 0;
  }

  .decision-total {
    position: absolute;
    bottom: 0;
    right: 0;
  }

  .decision-slash {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
  }

  .decision-modal-bg {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: rgba(0, 0, 0, 0.5);
  }

  .decision-modal {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: rgb(61, 61, 61);
    padding: 1rem;
    border-radius: 5px;
    width: 50%;
    max-width: 500px;
    text-align: center;
  }

  h1 {
    text-align: center;
  }

  main {
    display: flex;
    flex-direction: row;
    justify-content: center;
    width: 100%;
  }

  main div {
    margin: 0 1rem;
    width: 50%;
  }

  textarea {
    line-height: 21px;
    overflow-y: hidden;
    padding: 5px;
    border: 0;
    background: #474747;
    color: #fff;
    outline: none;
    resize: none;
    width: 100%;
    height: 50px;
  }

  @media (max-width: 768px) {
    h1 {
      font-size: 1.5rem;
    }

    p {
      word-wrap: break-word;
      font-size: 1rem;
    }

    main {
      flex-direction: column;
    }

    main div {
      width: 100%;
    }
  }
</style>
