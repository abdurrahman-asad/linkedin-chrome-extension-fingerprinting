## LinkedIn silently probes for 6000+ Chrome Extensions

This experiment is inspired by [this](https://github.com/mdp/linkedin-extension-fingerprinting) repo. While I was inspecting the script responsible for fingerprinting, I found out that the number of extensions in my case is almost double(6153). So the list might be different for different users based on region and/or other factors that I don't know.

You can see all these extensions in the [`list_of_extensions.csv`](list_of_extensions.csv) file.

### Setup
To check fingerprinting on your profile, below is a very simple script that you can run to get the extension names and URLs. If you run into any issues, check out the repo mentioned above for a better setup.

```javascript
import fs from "fs/promises";
const outputFile = "list_of_extensions.csv";

// Write CSV header
await fs.writeFile(outputFile, "id,title,url\n");

const tryFetch = async (url, retries = 2) => {
  try {
    const text = await (await fetch(url)).text();
    return text.match(/<title>(.*?)<\/title>/i)?.[1] || "";
  } catch (error) {
    if (retries > 0) {
      console.log(`Retrying ${url}...`);
      await new Promise(resolve => setTimeout(resolve, 500));
      return tryFetch(url, retries - 1);
    }
    console.error(`Failed to fetch ${url}:`, error.message);
    return "";
  }
};

const fetchTitle = async (id) => {
  let url = `https://chromewebstore.google.com/detail/${id}`;
  let title = await tryFetch(url);

  // if not found, chrome store redirects to home page
  if (!title || title === "Chrome Web Store") {
    url = `https://extpose.com/ext/${id}`;
    title = await tryFetch(url);
    // Clean Extpose format: "Name - id - Extpose"
    title = title.replace(/ - [a-z]+ - Extpose$/i, "").trim();
  } else {
    // Clean Chrome Web Store format: "Name - Chrome Web Store"
    title = title.replace(/ - Chrome Web Store$/i, "").trim();
  }

  title = title || "Title not found";
  console.log(`ID: ${id}, Title: ${title}`);
  return { id, title, url };
};

const batchSize = 50;
const idsArray = [...ids];

for (let i = 0; i < idsArray.length; i += batchSize) {
  const batch = idsArray.slice(i, i + batchSize);
  console.log(
    `Processing batch ${Math.floor(i / batchSize) + 1} of ${Math.ceil(idsArray.length / batchSize)}`,
  );

  const results = await Promise.all(batch.map(fetchTitle));
  const lines =
    results
      .map((r) => `${r.id},"${r.title.replace(/"/g, '""')}",${r.url}`)
      .join("\n") + "\n";
  await fs.appendFile(outputFile, lines);
}
```

Here, `extensions` is the array extracted from the source script. You can find it as follows:

- Open/Refresh LinkedIn, navigate to console in DevTools. You'll see many errors a few seconds after loading the page, and these errors will continue popping up for several minutes due to staggered fetches.
- Expand any error and go to the initiating line, the one at the bottom of the stack(refer to the video below). A few lines above will be the array with all the IDs and the respective files LinkedIn tries to fetch (in order to check for the presence of the extension).

https://github.com/user-attachments/assets/d635e208-428b-4fa8-b2fa-9d0d5d15f98d



Copy this array, store it in `extensions` and run the script to get `list_of_extensions.csv` file with the results.
