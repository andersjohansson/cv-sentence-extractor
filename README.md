# Common Voice Sentence Extractor

[Common Voice](https://voice.mozilla.org) is Mozilla's initiative to help teach machines how real people speak. For this we need to collect sentences that people can read out aloud on the website. Individual sentences can be submitted through the [Sentence Collector](https://common-voice.github.io/sentence-collector). This only can scale so far, so we also use automated tools to extract sentences from other sources.

Right now this tool supports extractions from the following sources:

* Wikipedia - max 3 sentences per articles

For a source to be added, the dataset needs to be vetted by Mozilla to check license compatibility. If you know about a good source, please start a topic on [Discourse](https://discourse.mozilla.org/c/voice/). Once it's been verified that a source can be used, check the "Adding another scrape target" further below.

## Setup

- [Rust Nightly](https://rustup.rs/) (follow the instructions and customize the install to select the `nightly` channel)

Note: as long as we're using the current `punkt` dependency, we need to use the Nightly version of Rust.

Clone this repo:

```
git clone https://github.com/Common-Voice/cv-sentence-extractor.git
```

### Wikipedia Extraction

You need to download the WikiExtractor:

```
git clone https://github.com/attardi/wikiextractor.git
```

## Extraction

### Extract Wikipedia

We can only extract at most 3 sentences per article.

1. Download the latest Wikipedia dataset [backup dump from Wikimedia](https://dumps.wikimedia.org/backup-index-bydb.html), select the one with `pages-articles-multistream` in its name.

Example (you can change "en" to your locale code)

```bash
wget https://dumps.wikimedia.org/enwiki/latest/enwiki-latest-pages-articles-multistream.xml.bz2
bzip2 -d enwiki-latest-pages-articles-multistream.xml.bz2
```

2. Use WikiExtractor to extract a dump (this might take a few hours)

```bash
cd wikiextractor
python WikiExtractor.py --json ../enwiki-latest-pages-articles-multistream.xml
```

*Important note: Please check the section about [creating a rules file](#using-language-rules) and [a blocklist](#create-a-blocklist-based-on-less-common-words) at this point, you might want to consider creating them and that process should happen before step 3.*

3. Scrape the sentences into a new file from the WikiExtractor output dir (this might take more than 6h to finish)

```bash
cd ../cv-sentence-extractor
cargo run -- extract -l en -d ../wikiextractor/text/ >> wiki.en.txt
```

*Tip: You don't need this last process to finish to start observing the output, wiki.en.txt should get a few thousands sentences in just a few minutes, and you can use that as a way to estimate the quality of the output early on and stop the process if you are not happy.*

## Using language rules

The following rules can be configured per language. Add a `<language>.toml` file in the `rules` directory to enable a new locale.

| Name   |      Description      |  Values | Default |
|--------|-----------------------|---------|---------|
| abbreviation_patterns |  Rust regex defining disallowed abbreviations. Sentences with abbreviations are generally not recommended since pronounciation is ambiguous. | Rust Regex Array | all abbreviations allowed
| allowed_symbols_regex |  Regex of allowed symbols or letters. Each character gets matched against this pattern. | String Array | not used
| broken_whitespace |  Array of broken whitespaces. This could for example disallow two spaces following each other | String Array | all types of whitespaces allowed
| disallowed_symbols |  Array of disallowed symbols or letters. Only used when allowed_symbols_regex is not set or is an empty String. | String Array | all symbols allowed
| disallowed_words |  Array of disallowed words | String Array | all words allowed
| even_symbols |  Symbols that always need an event count | Char Array | []
| matching_symbols |  Symbols that map to another | Array of matching configurations: each configuration is an Array of two values: `["match", "match"]`. See example below. | []
| max_word_count |  Maximum number of words in a sentence | integer | 14
| may_end_with_colon |  If a sentence can end with a : or not | boolean | false
| min_characters |  Minimum of character occurrences | integer | 0
| min_trimmed_length |  Minimum length of string after trimming | integer | 3
| min_word_count |  Minimum number of words in a sentence | integer | 1
| needs_letter_start |  If a sentence needs to start with a letter | boolean | true
| needs_punctuation_end |  If a sentence needs to end with a punctuation | boolean | false
| needs_uppercase_start |  If a sentence needs to start with an uppercase | boolean | false
| other_patterns |  Rust regex to disallow anything else | Rust Regex Array | all other patterns allowed
| quote_start_with_letter |  If a quote needs to start with a letter | boolean | true
| replacements |  Replaces abbreviations or other words according to configuration | Array of replacement configurations: each configuration is an Array of two values: `["search", "replacement"]`. See example below. | nothing gets replaced

### Example for `matching_symbols`

```
matching_symbols = [
  ["„", "“"],
  ["(", ")"],
  ["[", "]"]
]
```

This matches all occurrence of `„` with `“`, all occurrence of `(` with `)`, all occurrence of `[` with `]`.

```
Input: This is „a test“ and (another one)
Output: Valid

Input: This is (a test))
Output: Invalid
```

### Example for `replacements`

```
replacements = [
  ["test", "hi"],
  ["etc.", "et cetera"],
  ["foo", ""],
]
```

This replaces all occurrence of `test` with `hi`, all occurrence of `etc.` with `et cetera`, and removes all `foo`.

```
Input: I am a test etc.
Output: I am a hi et cetera

Input: I am foo test a test
Output: I am hi a hi
```

## Using disallowed words

In order to increase the quality of the final output, you might want to consider filtering out some words that are complex, too long or non-native.

You can do this by adding these words to the language rules file for your language under the `disallowed_words` setting.

If your list is too long, you can also place a `<language>.txt` file in the `rules/disallowed_words` directory to enable a new locale. Each word should be on a new line.

### Create a blocklist based on less common words

You can create a solid blocklist by generating a list of the less common words from your Wikipedia.

To do so, first you should create a full export with all Wikipedia sentences. Note that all processes below will take a while to execute.

After running step 1 and 2 from the `Usage` section above, run:

```bash
cd ../common-voice-wiki-scraper
cargo run -- extract -l en -d ../wikiextractor/text/ --no_check >> wiki.en.all.txt
```

Then you can use the cvtools scripts to generate a list of the word frequency:

```bash
cd  ..
git clone https://github.com/dabinat/cvtools/
cd cvtools
python3 ./word_usage.py -i ../common-voice-wiki-scraper/wiki.en.all.txt >> word_usage.en.txt
```

You will have to read the `word_usage.en.txt` file to decide where you should put the limit. Usually words with less than 80-60 repetitions are bad.

```bash
grep -i "80" ./word_usage.en.txt
```

Once you know the frequency limit, you can generate your blocklist by running:

```bash
python3 ./word_usage.py -i ../common-voice-wiki-scraper/wiki.en.all.txt --max-frequency 80 --show-words-only >> ../common-voice-wiki-scraper/src/rules/disallowed_words/en.txt
```

You can use also `--strip-by-apostrophe` which is handy for languages using `'` in their sentences to recognize more words.

When you run the scrapping in step 2 from the `Usage` section this list will automatically be used if present.

## Getting your rules/blocklist incorporated

In order to get your language rules and blocklist incorporated in this repo, you will need to create a Pull Request explaining the following:

- How many sentences did you get at the end?
- How did you create the blocklist file?
- Get at least 3 different native speakers (ideally linguistics) to review a random sample of 100-500 sentences and estimate the average error ratio and comment (or link their comment) in the PR.

Once we have your rules into the repo, we will be able to run the extraction from our side and incorporate the sentences into the Common Voice repo. But please take note that we have limited resources and we can't guarantee a specific date for us to run this process (we are looking into automating it).

## Adding another scrape target

If you find a new open data source that provides a lot of sentences ([Example](https://discourse.mozilla.org/t/using-the-europarl-dataset-with-sentences-from-speeches-from-the-european-parliament/50184/36)), we suggest to not go through through the Sentence Collector but rather adding a scrape target here. Before you do so, let's discuss it on [Discourse](https://discourse.mozilla.org/c/voice/) first!

* In `app.rs`, add a new extraction command - same arguments as the `extract` task, but with a better - more descriptive - name identifying your data source
* In `loader.rs` write your own loader according to the data structure - the data structure should be fairly simple, you might need to consider writing a separate script to fetch and prepare the sentences first (as we do with the WikiExtractor for Wikipedia)
* In `app.rs` add a new `else if` in the `start` function to generate your config and start the extraction, passing your own custom loader you wrote

## Automatic extraction

Currently the following data sources are available for automatic extraction:

* Wikipedia

### On every Pull Request

On every PR we will [trigger a sample sentence extraction](https://discourse.mozilla.org/t/scraper-automatic-sample-sentences-extracted-in-pull-request/55217/3) which can be used for verification.

### On pushes to master

On every push to master, we will run a full sentence extraction on the specified language if the commit message includes `--full-wiki-extraction=<language>` at the end of the message.
