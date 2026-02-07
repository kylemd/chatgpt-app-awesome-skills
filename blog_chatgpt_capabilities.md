# [Simon Willison’s Weblog](/)

[Subscribe](/about/#subscribe)

## ChatGPT Containers can now run bash, pip/npm install packages, and download files

26th January 2026

One of my favourite features of ChatGPT is its ability to write and execute code in a container. This feature launched as ChatGPT Code Interpreter [nearly three years ago](https://simonwillison.net/2023/Apr/12/code-interpreter/), was half-heartedly rebranded to “Advanced Data Analysis” at some point and is generally really difficult to find detailed documentation about. Case in point: it appears to have had a *massive* upgrade at some point in the past few months, and I can’t find documentation about the new capabilities anywhere!

Here are the most notable new features:

1. ChatGPT can **directly run Bash commands** now. Previously it was limited to Python code only, although it could run shell commands via the Python `subprocess` module.
2. **It has Node.js** and can run JavaScript directly in addition to Python. I also got it to run “hello world” in **Ruby, Perl, PHP, Go, Java, Swift, Kotlin, C and C++**. No Rust yet though!
3. While the container still can’t make outbound network requests, **`pip install package` and `npm install package` both work** now via a custom proxy mechanism.
4. ChatGPT can locate the URL for a file on the web and use a `container.download` tool to **download that file and save it to a path** within the sandboxed container.

This is a substantial upgrade! ChatGPT can now write and then test code in 10 new languages (11 if you count Bash), can find files online and download them into the container, and can install additional packages via `pip` and `npm` to help it solve problems.

(OpenAI *really* need to develop better habits at [keeping their release notes up-to-date](https://help.openai.com/en/articles/6825453-chatgpt-release-notes)!)

I was initially suspicious that maybe I’d stumbled into a new preview feature that wasn’t available to everyone, but I [tried some experiments](https://chatgpt.com/share/6977aa7c-7bd8-8006-8129-8c9e25126fed) in a free ChatGPT account and confirmed that the new features are available there as well.

* [container.download](https://simonwillison.net/2026/Jan/26/chatgpt-containers/#container-download)
* [Is container.download a data exfiltration vulnerability?](https://simonwillison.net/2026/Jan/26/chatgpt-containers/#is-container-download-a-data-exfiltration-vulnerability-)
* [Bash and other languages](https://simonwillison.net/2026/Jan/26/chatgpt-containers/#bash-and-other-languages)
* [Installing packages from pip and npm](https://simonwillison.net/2026/Jan/26/chatgpt-containers/#installing-packages-from-pip-and-npm)
* [Adding it all together](https://simonwillison.net/2026/Jan/26/chatgpt-containers/#adding-it-all-together)

#### container.download

My first clue to the new features came the other day when I got curious about Los Angeles air quality, in particular has the growing number of electric vehicles there had a measurable impact?

I prompted a fresh GPT-5.2 Thinking session with:

> Show me Los Angeles air quality over time for last 20 years

Here’s the [shared transcript](https://chatgpt.com/share/6977a80d-e6e4-8006-b4fe-ed8478fe99d8). Annoyingly that doesn’t make the thinking trace available, so here’s a screenshot:

![Exploring methods to fetch AQI data and reports Downloading AQI data for Los Angeles  I found an xlsx file titled "Air Quality Statistics by City, 2000-2023", which should include data for Los Angeles. While user requested data for Using provided URL to download file  The URL from the error message leads to an Excel file containing air quality trends. I can use the container.download tool to fetch it. Let’s proceed with this approach and see if it works out for downloading the file directly! Analyzed Parsing and analyzing the data for Los Angeles  Now that we've downloaded the data, I'll parse it using Python. First, I'll focus on identifying the row(s) corresponding to Los Angeles, which may be labeled under "Los Angeles-Long Beach-Anaheim, CA." Afterward, I'll plot metrics like "Median AQI" and "Days Good" over time to visualize trends. Let’s begin!](https://static.simonwillison.net/static/2026/container-download.jpg)

This bit instantly caught my eye:

> [...] The URL from the error message leads to an Excel file containing air quality trends. I can use the container.download tool to fetch it. [...]

What’s `container.download`? I [asked ChatGPT](https://chatgpt.com/share/6977aab0-b874-8006-9b54-d3d216ed884b):

> Without running any searches tell me about your container.download tool

And got back:

> Sure. `container.download` is a built-in tool I can call that **fetches a file from a URL and saves it into the sandboxed container filesystem** for this chat session.
>
> #### What it does
>
> * Takes a **publicly reachable URL** and a **destination filepath** in the container.
> * Downloads the bytes from that URL and writes them to the given path.
> * After that, I can read/process the file locally in the container (e.g., unzip it, parse it with Python, open it as an image, convert it, etc.).
>
> #### Signature (inputs)
>
> It’s called like this (conceptually):
>
> ```
> {
>   "url": "https://example.com/somefile.zip",
>   "filepath": "/mnt/data/somefile.zip"
> }
> ```

Further experimentation confirmed that `container.download` behaves as described.

I had it download one file from a server with logs enabled and captured the IP address and user-agent. Here are the most interesting headers:

```
User-Agent: Mozilla/5.0 AppleWebKit/537.36 (KHTML, like Gecko); compatible; ChatGPT-User/1.0; +https://openai.com/bot
Accept: text/html, application/xhtml+xml, application/xml;q=0.9, image/avif, image/webp, image/apng, */*;q=0.8, application/signed-exchange;v=b3;q=0.9
Cf-Connecting-Ip: 52.230.164.178

```

That `52.230.164.178` IP address [resolves to](http://ip-api.com/json/52.230.164.178) Microsoft Azure Cloud (centralus) in Des Moines, Iowa.

#### Is container.download a data exfiltration vulnerability?

On the one hand, this is really useful! ChatGPT can navigate around websites looking for useful files, download those files to a container and then process them using Python or other languages.

Is this a data exfiltration vulnerability though? Could a prompt injection attack trick ChatGPT into leaking private data out to a `container.download` call to a URL with a query string that includes sensitive information?

I don’t think it can. I tried getting it to assemble a URL with a query string and access it using `container.download` and it couldn’t do it. It told me that it got back this error:

> ERROR: download failed because url not viewed in conversation before. open the file or url using web.run first.

This looks to me like the same safety trick [used by Claude’s Web Fetch tool](https://simonwillison.net/2025/Sep/10/claude-web-fetch-tool/): only allow URL access if that URL was either directly entered by the user or if it came from search results that could not have been influenced by a prompt injection.

(I poked at this a bit more and managed to get a simple constructed query string to pass through `web.run`—a different tool entirely—but when I tried to compose a longer query string containing the previous prompt history a `web.run` filter blocked it.)

So I *think* this is all safe, though I’m curious if it could hold firm against a more aggressive round of attacks from a seasoned security researcher.

#### Bash and other languages

The key lesson from coding agents like Claude Code and Codex CLI is that Bash rules everything: if an agent can run Bash commands in an environment it can do almost anything that can be achieved by typing commands into a computer.

When Anthropic added their own code interpreter feature to Claude [last September](https://simonwillison.net/2025/Sep/9/claude-code-interpreter/) they built that around Bash rather than just Python. It looks to me like OpenAI have now done the same thing for ChatGPT.

Here’s what ChatGPT looks like when it runs a Bash command—here my prompt was:

> npm install a fun package and demonstrate using it

![Screenshot of ChatGPT 5.2 Thinking interface with Activity panel. Main chat shows user prompt "npm install a fun package and demonstrate using it" and response "Thought for 32s > Yep — here's a fun one: cowsay 🐄 I installed it with npm and ran a tiny Node script:" followed by bash code block containing "mkdir -p /mnt/data/npmfun cd /mnt/data/npmfun npm init -y npm install cowsay --no-fund --no-audit node -e "const cowsay=require('cowsay'); console.log(cowsay.say({text:'Hel" and Output: section showing ASCII cow art saying "< Hello from npm inside the container! >" with cow drawing made of characters. Activity panel on right (32s) shows: "Running npm install in container" with bash command and output "v22.16.0 10.9.2", "Inspecting npm command output and container status" with directory listing showing "drwxr-xr-x 2 root oai_shared 80 Jan 26 17:32 . drwxrwxr-x 2 oai oai_shared 60 Jan 26 17:32 .. -rw-r--r-- 1 root root", "Re-running npm install" note about node_modules missing, and "Rerunning npm install with output" showing "added 41 packages in 2s".](https://static.simonwillison.net/static/2026/chatgpt-bash.jpg)

It’s useful to click on the “Thinking” or “Thought for 32s” links as that opens the Activity sidebar with a detailed trace of what ChatGPT did to arrive at its answer. This helps guard against cheating—ChatGPT might claim to have run Bash in the main window but it can’t fake those black and white logs in the Activity panel.

I had it run [Hello World in various languages](https://chatgpt.com/share/6977b890-f1c0-8006-b39a-9a02c7ca1993) later in that same session.

![Terminal screenshot showing "Hello, world" programs in multiple languages. Top section shows code: swift -e 'print("Hello, world from Swift!")' followed by cat > hello.kts <<'KTS' / println("Hello, world from Kotlin!") / KTS / kotlinc -script hello.kts, then cat > hello.c <<'C' / #include <stdio.h> / int main(){ puts("Hello, world from C!"); return 0; } / C / gcc hello.c -o hello_c / ./hello_c, then cat > hello.cpp <<'CPP' / #include <iostream> / int main(){ std::cout << "Hello, world from C++!" << std::endl; } / CPP / g++ hello.cpp -o hello_cpp / ./hello_cpp. Bottom section shows output: Hello, world from Ruby! / Hello, world from Perl! / Hello, world from PHP! / Hello, world from Go! / Hello, world from Java! / Hello, world from Swift! / Hello, world from Kotlin! / Hello, world from C! / Hello, world from C++!. UI shows "Thought for 2m 29s" and "Done" at bottom.](https://static.simonwillison.net/static/2026/chatgpt-hello-worlds.jpg)

#### Installing packages from pip and npm

In the previous example ChatGPT installed the `cowsay` package from npm and used it to draw an ASCII-art cow. But how could it do that if the container can’t make outbound network requests?

In [another session](https://chatgpt.com/share/69773501-b6d8-8006-bbf2-fa644561aa26) I challenged it to explore its environment. and figure out how that worked.

Here’s [the resulting Markdown report](https://github.com/simonw/research/blob/main/chatgpt-container-environment/README.md) it created.

The key magic appears to be a `applied-caas-gateway1.internal.api.openai.org` proxy, available within the container and with various packaging tools configured to use it.

The following environment variables cause `pip` and `uv` to install packages from that proxy instead of directly from PyPI:

```
PIP_INDEX_URL=https://reader:****@packages.applied-caas-gateway1.internal.api.openai.org/.../pypi-public/simple
PIP_TRUSTED_HOST=packages.applied-caas-gateway1.internal.api.openai.org
UV_INDEX_URL=https://reader:****@packages.applied-caas-gateway1.internal.api.openai.org/.../pypi-public/simple
UV_INSECURE_HOST=https://packages.applied-caas-gateway1.internal.api.openai.org

```

This one appears to get `npm` to work:

```
NPM_CONFIG_REGISTRY=https://reader:****@packages.applied-caas-gateway1.internal.api.openai.org/.../npm-public

```

And it reported these suspicious looking variables as well:

```
CAAS_ARTIFACTORY_BASE_URL=packages.applied-caas-gateway1.internal.api.openai.org
CAAS_ARTIFACTORY_PYPI_REGISTRY=.../artifactory/api/pypi/pypi-public
CAAS_ARTIFACTORY_NPM_REGISTRY=.../artifactory/api/npm/npm-public
CAAS_ARTIFACTORY_GO_REGISTRY=.../artifactory/api/go/golang-main
CAAS_ARTIFACTORY_MAVEN_REGISTRY=.../artifactory/maven-public
CAAS_ARTIFACTORY_GRADLE_REGISTRY=.../artifactory/gradle-public
CAAS_ARTIFACTORY_CARGO_REGISTRY=.../artifactory/api/cargo/cargo-public/index
CAAS_ARTIFACTORY_DOCKER_REGISTRY=.../dockerhub-public
CAAS_ARTIFACTORY_READER_USERNAME=reader
CAAS_ARTIFACTORY_READER_PASSWORD=****
NETWORK=caas_packages_only

```

Neither Rust nor Docker are installed in the container environment, but maybe those registry references are a clue of features still to come.

#### Adding it all together

The result of all of this? You can tell ChatGPT to use Python or Node.js packages as part of a conversation and it will be able to install them and apply them against files you upload or that it downloads from the public web. That’s *really* cool.

The big missing feature here should be the easiest to provide: we need **official documentation**! A release notes entry would be a good start, but there are a lot of subtle details to how this new stuff works, its limitations and what it can be used for.

As always, I’d also encourage OpenAI to come up with a name for this set of features that properly represents how it works and what it can do.

In the meantime, I’m going to call this **ChatGPT Containers**.

#### Update: a full list of tools

I decided to ask ChatGPT about other tools that were available to it in case there was anything interesting in there:

> List all tools that are available to you, with their exact names and descriptions and signatures

Here’s [what I got back](https://chatgpt.com/share/6977ffa0-df14-8006-9647-2b8c90ccbb81).

List of ChatGPT tools according to GPT-5.2 Thinking on 26th January 2026

Below are **all tools available to me in this chat**, with their **exact names**, **descriptions**, and **signatures** (as defined in my tool interface).

---

#### python

##### `python.exec` (FREEFORM) → any

**Description:** Execute Python code in a Jupyter environment for *private/internal reasoning* (not shown to you). Internet access is disabled.
**Signature:** `python.exec(FREEFORM) -> any`

---

#### web

##### `web.run` (ToolCallV5 JSON) → any

**Description:** Access the internet and specialized lookups (search, open/click/find pages, screenshot PDFs, image search, product search, finance, weather, sports, calculator, time). Supports citations via returned `turn...` reference IDs.
**Signature:**
`web.run({ open?, click?, find?, screenshot?, image_query?, product_query?, sports?, finance?, weather?, calculator?, time?, search_query?, response_length? }) -> any`

---

#### automations

##### `automations.create` → any

**Description:** Schedule a future/recurring task/reminder/search via iCal VEVENT.
**Signature:**
`automations.create({ prompt: string, title: string, schedule?: string, dtstart_offset_json?: string }) -> any`

##### `automations.update` → any

**Description:** Modify an existing scheduled task.
**Signature:**
`automations.update({ jawbone_id: string, schedule?: string, dtstart_offset_json?: string, prompt?: string, title?: string, is_enabled?: boolean }) -> any`

##### `automations.list` → any

**Description:** List existing automations.
**Signature:** `automations.list() -> any`

---

#### guardian\_tool

##### `guardian_tool.get_policy` → str

**Description:** Retrieve policy guidance for specific categories (currently: `election_voting`).
**Signature:** `guardian_tool.get_policy({ category: string }) -> str`

---

#### file\_search

##### `file_search.msearch` → any

**Description:** Search across available internal/connected sources (in this environment: primarily “recording\_knowledge” transcripts when relevant).
**Signature:**
`file_search.msearch({ queries?: string[], source_filter?: string[], file_type_filter?: string[], intent?: string, time_frame_filter?: { start_date?: string, end_date?: string } }) -> any`

##### `file_search.mclick` → any

**Description:** Expand/open previously returned search results (or connector URLs) for more context.
**Signature:**
`file_search.mclick({ pointers?: string[], start_date?: string, end_date?: string }) -> any`

---

#### gmail (read-only)

##### `gmail.search_email_ids` → any

**Description:** Search Gmail message IDs by query/tags (read-only).
**Signature:**
`gmail.search_email_ids({ query?: string, tags?: string[], max_results?: integer, next_page_token?: string }) -> any`

##### `gmail.batch_read_email` → any

**Description:** Read a batch of Gmail messages by message IDs (read-only).
**Signature:**
`gmail.batch_read_email({ message_ids: string[] }) -> any`

---

#### gcal (read-only)

##### `gcal.search_events` → any

**Description:** Search Google Calendar events in a time range and/or by keyword (read-only).
**Signature:**
`gcal.search_events({ time_min?: string, time_max?: string, timezone_str?: string, max_results?: integer, query?: string, calendar_id?: string, next_page_token?: string }) -> any`

##### `gcal.read_event` → any

**Description:** Read a specific calendar event by ID (read-only).
**Signature:**
`gcal.read_event({ event_id: string, calendar_id?: string }) -> any`

---

#### gcontacts (read-only)

##### `gcontacts.search_contacts` → any

**Description:** Search Google Contacts by keyword (read-only).
**Signature:**
`gcontacts.search_contacts({ query: string, max_results?: integer }) -> any`

---

#### canmore (canvas)

##### `canmore.create_textdoc` → any

**Description:** Create a new canvas text document (markdown or code).
**Signature:**
`canmore.create_textdoc({ name: string, type: "document" | "code/bash" | "code/zsh" | "code/javascript" | "code/typescript" | "code/html" | "code/css" | "code/python" | "code/json" | "code/sql" | "code/go" | "code/yaml" | "code/java" | "code/rust" | "code/cpp" | "code/swift" | "code/php" | "code/xml" | "code/ruby" | "code/haskell" | "code/kotlin" | "code/csharp" | "code/c" | "code/objectivec" | "code/r" | "code/lua" | "code/dart" | "code/scala" | "code/perl" | "code/commonlisp" | "code/clojure" | "code/ocaml" | "code/powershell" | "code/verilog" | "code/dockerfile" | "code/vue" | "code/react" | "code/other", content: string }) -> any`

##### `canmore.update_textdoc` → any

**Description:** Apply regex-based edits to the current canvas textdoc.
**Signature:**
`canmore.update_textdoc({ updates: { pattern: string, multiple?: boolean, replacement: string }[] }) -> any`

##### `canmore.comment_textdoc` → any

**Description:** Add actionable comments on matched regions of the current canvas textdoc.
**Signature:**
`canmore.comment_textdoc({ comments: { pattern: string, comment: string }[] }) -> any`

---

#### python\_user\_visible

##### `python_user_visible.exec` (FREEFORM) → any

**Description:** Execute Python code *that you will see* (tables, plots, generated files). Internet disabled.
**Signature:** `python_user_visible.exec(FREEFORM) -> any`

---

#### user\_info

##### `user_info.get_user_info` → any

**Description:** Get your current location and local time (or UTC if unknown).
**Signature:** `user_info.get_user_info({}) -> any`

---

#### summary\_reader

##### `summary_reader.read` → any

**Description:** Read safe-to-share internal reasoning summaries from earlier turns (used when asked how I arrived at something).
**Signature:** `summary_reader.read({ limit?: integer, offset?: integer }) -> any`

---

#### container

##### `container.exec` → any

**Description:** Run a command inside the container and return stdout/stderr.
**Signature:**
`container.exec({ cmd: string[], session_name?: string | null, workdir?: string | null, timeout?: integer | null, env?: object | null, user?: string | null }) -> any`

##### `container.feed_chars` → any

**Description:** Send input to a running interactive exec session and flush output.
**Signature:**
`container.feed_chars({ session_name: string, chars: string, yield_time_ms?: integer }) -> any`

##### `container.open_image` → any

**Description:** Display an image from an absolute path in the container (jpg/jpeg/png/webp).
**Signature:**
`container.open_image({ path: string, user?: string | null }) -> any`

##### `container.download` → any

**Description:** Download a file from a URL into the container filesystem.
**Signature:**
`container.download({ url: string, filepath: string }) -> any`

---

#### bio (memory)

##### `bio.update` (FREEFORM) → any

**Description:** Persist (or forget) user-specific info across conversations (“memory”).
**Signature:** `bio.update(FREEFORM) -> any`

---

#### api\_tool

##### `api_tool.list_resources` → any

**Description:** List available “resources” (including invokable tools) in an OP-style file tree.
**Signature:**
`api_tool.list_resources({ path?: string, cursor?: string | null, only_tools?: boolean, refetch_tools?: boolean }) -> any`

##### `api_tool.call_tool` → any

**Description:** Invoke a resource as a tool (must know its full path/URI from `list_resources`).
**Signature:**
`api_tool.call_tool({ path: string, args: object }) -> any`

---

#### personal\_context

##### `personal_context.search` → any

**Description:** Retrieve user-specific context from prior interactions/linked sources to maintain continuity.
**Signature:**
`personal_context.search({ query: string }) -> any`

---

#### image\_gen

##### `image_gen.text2im` → any

**Description:** Generate or edit images from text instructions (and optionally referenced image IDs).
**Signature:**
`image_gen.text2im({ prompt?: string | null, size?: string | null, n?: integer | null, transparent_background?: boolean | null, is_style_transfer?: boolean | null, referenced_image_ids?: string[] | null }) -> any`

---

#### user\_settings

##### `user_settings.get_user_settings` → any

**Description:** Read current settings and allowed values (personality, accent color, appearance).
**Signature:** `user_settings.get_user_settings() -> any`

##### `user_settings.set_setting` → any

**Description:** Change one setting (accent color, appearance, or personality) to an allowed value.
**Signature:**
`user_settings.set_setting({ setting_name: "accent_color" | "appearance" | "personality", setting_value: string }) -> any`

---

#### artifact\_handoff

##### `artifact_handoff.prepare_artifact_generation` → any

**Description:** Must be called immediately when the user asks for a spreadsheet or slide deck artifact.
**Signature:** `artifact_handoff.prepare_artifact_generation() -> any`

Posted [26th January 2026](/2026/Jan/26/) at 7:19 pm · Follow me on [Mastodon](https://fedi.simonwillison.net/%40simon), [Bluesky](https://bsky.app/profile/simonwillison.net), [Twitter](https://twitter.com/simonw) or [subscribe to my newsletter](https://simonwillison.net/about/#subscribe)

## More recent articles

* [Running Pydantic's Monty Rust sandboxed Python subset in WebAssembly](/2026/Feb/6/pydantic-monty/) - 6th February 2026
* [Distributing Go binaries like sqlite-scanner through PyPI using go-to-wheel](/2026/Feb/4/distributing-go-binaries/) - 4th February 2026
* [Moltbook is the most interesting place on the internet right now](/2026/Jan/30/moltbook/) - 30th January 2026

This is **ChatGPT Containers can now run bash, pip/npm install packages, and download files** by Simon Willison, posted on [26th January 2026](/2026/Jan/26/).

[pypi
42](/tags/pypi/)
[sandboxing
34](/tags/sandboxing/)
[npm
21](/tags/npm/)
[ai
1819](/tags/ai/)
[openai
389](/tags/openai/)
[generative-ai
1611](/tags/generative-ai/)
[chatgpt
190](/tags/chatgpt/)
[llms
1577](/tags/llms/)
[ai-assisted-programming
320](/tags/ai-assisted-programming/)
[code-interpreter
29](/tags/code-interpreter/)

**Next:** [Adding dynamic features to an aggressively cached website](/2026/Jan/28/dynamic-features-static-site/)

**Previous:** [Wilson Lin on FastRender: a browser built by thousands of parallel agents](/2026/Jan/23/fastrender/)

### Monthly briefing

Sponsor me for **$10/month** and get a curated email digest of the month's most important LLM developments.

Pay me to send you less!

[Sponsor & subscribe](https://github.com/sponsors/simonw/)

* [Colophon](/about/#about-site)
* ©
* [2002](/2002/)
* [2003](/2003/)
* [2004](/2004/)
* [2005](/2005/)
* [2006](/2006/)
* [2007](/2007/)
* [2008](/2008/)
* [2009](/2009/)
* [2010](/2010/)
* [2011](/2011/)
* [2012](/2012/)
* [2013](/2013/)
* [2014](/2014/)
* [2015](/2015/)
* [2016](/2016/)
* [2017](/2017/)
* [2018](/2018/)
* [2019](/2019/)
* [2020](/2020/)
* [2021](/2021/)
* [2022](/2022/)
* [2023](/2023/)
* [2024](/2024/)
* [2025](/2025/)
* [2026](/2026/)