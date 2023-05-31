<script lang="ts">
  let quotes: string;
  let result: string;
  let similarityPercentage: number = 50;

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

  const submit = (e: SubmitEvent): void => {
    const tokens = tokeniseQuotes(quotes);
    if (!tokens) {
      result = '';
      return;
    }
    const uniqueTokens: string[] = [];
    for (let i = 0; i < tokens.length; i++) {
      const token = tokens[i];
      let isUnique = true;
      for (let j = 0; j < uniqueTokens.length; j++) {
        const otherToken = tokens[j];
        if (i === j) continue;
        const sim = similarity(token, otherToken);
        console.log(token, otherToken, sim);

        if (sim * 100 >= similarityPercentage) {
          isUnique = false;
          break;
        }
      }
      if (isUnique) {
        uniqueTokens.push(token);
      }
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
        />%
      </p>
      <p>
        If a comparison of two quotes is more than, or equal to {similarityPercentage}
        per cent similar, it will not be added to the results.
      </p>
      <textarea bind:value={quotes} on:input={textAreaChange} />

      <br />
      <br />
      <input type="submit" value="Remove duplicates" />
    </form>
  </div>
  <div>
    <p>Removed duplicates:</p>
    <textarea bind:value={result} readonly id="result" />
  </div>
</main>
<p style="position: fixed; left: 0; bottom: 0; padding: 0; margin: 0;">
  Hi Sam :)
</p>

<style>
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
</style>
