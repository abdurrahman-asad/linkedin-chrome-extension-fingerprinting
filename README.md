## LinkedIn silently probes for 6000+ Chrome Extensions

This experiment is inspired by [this](https://github.com/mdp/linkedin-extension-fingerprinting) repo. While I was inspecting the script responsible for fingerprinting, I found out the number of extensions in my case is almost double. So the list might be different for different users based on region and/or other factors that I don't know.

You can see all these extensions in the [`list_of_extensions.csv`](list_of_extensions.csv) file.

If you want to check it in your case, below is a very simple script(why over-engineer when it works) that I ran to get the extension names. If you run into any issues, check out the repo mentioned above for a better setup.

```javascript
import fs from "fs/promises";
const outputFile = "extension_titles.jsonl";

const tryFetch = async (url) => {
  const text = await (await fetch(url)).text();
  return text.match(/<title>(.*?)<\/title>/i)?.[1] || "";
};

const fetchTitle = async (id) => {
  let title = await tryFetch(`https://chromewebstore.google.com/detail/${id}`);
  // if not found, chrome store redirects to home page
  if (!title || title === "Chrome Web Store") {
    title = await tryFetch(`https://extpose.com/ext/${id}`);
  }

  title = title || "Title not found";
  console.log(`ID: ${id}, Title: ${title}`);
  return { id, title };
};

const ids = new Set(extensions.map((e) => e.id));
const batchSize = 50;
const idsArray = [...ids];

for (let i = 0; i < idsArray.length; i += batchSize) {
  const batch = idsArray.slice(i, i + batchSize);
  console.log(
    `Processing batch ${Math.floor(i / batchSize) + 1} of ${Math.ceil(idsArray.length / batchSize)}`,
  );

  const results = await Promise.all(batch.map(fetchTitle));
  const lines = results.map((r) => JSON.stringify(r)).join("\n") + "\n";
  await fs.appendFile(outputFile, lines);
}
```

Here `extensions` is the array extracted from the source script. You can find it as follows:

- Open/Refresh LinkedIn, navigate to console in DevTools. You'll see many errors a few seconds after loading the page, and these errors will continue popping up for several minutes due to staggered fetches.
- Expand any error and go to the initiating line, the one at the bottom of the stack(refer to the video below). A few lines above will be the array with all the IDs and the respective files LinkedIn tries to fetch (in order to check for presence of the extension).

Copy this array, store it in `extensions` and run the script to get `extension_titles.jsonl` file with result.