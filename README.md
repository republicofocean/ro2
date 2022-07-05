# ro2
This is the root and inception of the Republic of Ocean
----

<h1 align="center">Republic of Ocean</h1>

> 🏄‍♀️ At the intersection of Audio, Blockchain, Community & Data.
> https://www.republicofocean.com/

[![Build Status](https://travis-ci.com/ro2/commons.svg?branch=master)](https://travis-ci.com/oceanprotocol/commons)
[![code style: prettier](https://img.shields.io/badge/code_style-prettier-7b1173.svg?style=flat-square)](https://github.com/prettier/prettier)
[![js oceanprotocol](https://img.shields.io/badge/js-oceanprotocol-7b1173.svg)](https://github.com/oceanprotocol/eslint-config-oceanprotocol)

<img width="1218" alt="Republic of Ocean" src="https://serving.photos.photobox.com/30031721d7b04c9f68995bc9bcf2240ff05d55eda919a75303c6b6cec171dab393d75876.jpg">

---

<h3 align="center">🦑🦑🦑<br />This project is deployed under <a href="https://www.republicofocean.com/">republicofocean.com</a> and can be used there. Feel free to <a href="https://github.com/ro2/commons/issues">report any issues</a> you encounter.<br />🦑🦑🦑</h3>

---

If you're a developer and want to contribute to, or want to utilize this marketplace's code in your projects, then keep on reading.

- [🏄 Get Started](#-get-started)
- [👩‍🔬 Testing](#-testing)
  - [Unit Tests](#unit-tests)
  - [End-to-End Integration Tests](#end-to-end-integration-tests)
- [✨ Code Style](#-code-style)
- [🛳 Production](#-production)
- [⬆️ Releases](#️-releases)
- [📜 Changelog](#-changelog)
- [🎁 Contributing](#-contributing)
- [🏛 License](#-license)

## 🏄 Get Started

This repo can be deployed on AWS as a state machine, it uses a serverless architecture.


1. Navigate to the **RSS Step Functions State Machine** and click on the `RssStateMachineUrl`

1. In the Step Functions console, create an execution using the **Start Execution** button and provide the input payload like below:

	```json
	{
	  "PodcastName": "DAO Podcast",
	  "rss": "https://feeds.simplecast.com/GNuMtYW3",
	  "maxEpisodesToProcess": 10,
	  "dryrun": "FALSE"
	}
	```

	> Note: You can choose a different podcast feed by changing the rss  input parameter.
	> 
	> The `maxEpisodesToProcess` input parameter lets you control the number of episodes to process from this feed. 
	> 
	> 	The `dryrun` flag will test the state machine without calling the AI functions. Leave is to FALSE to fully process the podcast.


Wait for workflow execution to complete. Amazon Transcribe can take about 10-15 minutes to process the 10 episodes (note that there's a default soft limit of 10 concurrent jobs that may be increased per request). Note that you will be able to see results appear in the ElasticSearch index as soon as some executions of the child workflow **EpisodeStateMachine** completes, even while the parent **RssStateMachine** is still waiting on the rest of the epsidoes to finish. 



### Lambda functions

#### RSS Feed Step Function State Machine Lambda functions
 
* **processPodcastRss**: Downloads the RSS file and parses it to determine the episodes to download. This function also leverages Amazon Comprehend's [**entity extraction**] feature for 2 use cases:

	* To compute an estimate of the number of speakers in each episode. We do this by using Amazon Comprehend to find people's names in each episode's abstract. We find that many podcast hosts like to mention their guest speakers’ names in the abstract. This helps us later when we use Amazon Transcribe to break out the transcription into multiple speakers. If no names are found in the abstract, we will assume the episode has a single speaker. 

	* To build a domain-specific custom vocabulary list. If a podcast is about AWS, you will hear lots of expressions unique to the specific domain (e.g., EC2, S3) that are completely different from expressions found in a podcast about Web3 (e.g., Dapps, DEFI). Providing a custom vocabulary list to Amazon Transcribe can help guide the service in identifying an audio segment that sounds like “easy too” to its actual meaning “EC2.” In this blog post, we automatically generate the custom vocabulary list by using the named entities extracted from episode abstracts to make Amazon Transcribe more domain aware. Keep in mind that this approach may not cover all jargon that could appear in the transcripts. To get more accurate transcriptions, you can complement this approach by drafting a list of common domain-specific terms so that you can construct a custom vocabulary list for Amazon Transcribe. 


* **createTranscribeVocabulary**: Creates a [**custom vocabulary**] for the Amazon Transcribe jobs so it will better understand when an AWS/tech jargon is mentioned. The custom vocabulary is created using the method mentioned above. 
* **monitorTranscribeVocabulary**: Polls Amazon Transcribe to determine if the custom vocabulary creation has completed.
* **createElasticsearchIndex**: Creates [**index mappings**] in ElasticSearch
* **processPodcastItem**: Creates a child state machine execution for each episode while maintaining a maximum of 10 concurrent child processes. This function keeps track of how many processes are active and throttles the downstream calls once the maximum is hit. Amazon S3 is used to store additional state about each episode. 
* **deleteTranscribeVocabulary**: Cleans up the custom vocabulary after the processing of all episodes is complete. Note that we added this step to minimize artifacts that stay around in your account after you run the demo application. However, when you build your own apps with Amazon Transcribe, you should consider keeping the custom vocabulary around for future processing jobs.

#### Episode Step Function State Machine Lambda functions

* **downloadPodcast**: Downloads the podcast from the publisher and stages it in S3 for further processing.
* **podcastTranscribe**: Makes the call to Amazon Transcribe to create the transcription job. Notice how we pass in parameters extracted from previous steps, such as the custom vocabulary to use and number of speakers for the episode. 
* **checkTranscript**: Polls the transcription job for status. Returns the status and the step function will retry of the job is in progress.
* **processTranscriptionParagraph**: This is the most complicated function in the application. You extract the transcription data from transcribe and break it out into paragraphs. The paragraphs are broken by speaker, punctuation, or a maximum length. The output of this function is a file that contains all the paragraphs in the transcription job as well as the start time of when the phrases was spoken in the audio file and the speaker the paragraph is attributed to.
* **processTranscriptionFullText**: This function contains similar logic to **processTranscriptionParagraph**, but the output is a full text transcription in a readable format. 
* **UploadToElasticsearch**: Parses the output of the previous steps and performs a bulk load of the indexes into the Elasticsearch cluster. The connection to Elasticsearch uses a SigV4 signature to perform IAM based authentication into the cluster.


```
                                        
## 👩‍🔬 Testing

Test suite is setup with [Jest](https://jestjs.io) and [react-testing-library](https://github.com/kentcdodds/react-testing-library) for unit testing, and [Cypress](https://www.cypress.io) for integration testing.

```

### End-to-End Integration Tests

To run all integration tests in headless mode, run:

```bash
npm run test:e2e
```

This will automatically spin up all required resources to run the integrations tests, and then run them.

You can also use the UI of Cypress to run and inspect the integration tests locally:

```bash
npm run cypress:open
```

## ✨ Code Style

For linting and auto-formatting you can use from the root of the project:

```bash
# auto format all ts & css with eslint
npm run lint

# auto format all ts & css with prettier, taking all configs into account
npm run format
```

## 🛳 Production

To create a production build of both, the client and the server, run from the root of the project:

```bash
npm run build
```

Builds the client for production to the `./client/build` folder, and the server into the `./server/dist` folder.

## ⬆️ Releases

From a clean `master` branch you can run any release task doing the following:

- bumps the project version in `package.json`, `client/package.json`, `server/package.json`
- auto-generates and updates the CHANGELOG.md file from commit messages
- creates a Git tag
- commits and pushes everything
- creates a GitHub release with commit messages as description

You can execute the script using {major|minor|patch} as first argument to bump the version accordingly:

- To bump a patch version: `npm run release`
- To bump a minor version: `npm run release minor`
- To bump a major version: `npm run release major`

By creating the Git tag with these tasks, Travis will trigger a new Kubernetes live deployment automatically, after a successful tag build.

For the GitHub releases steps a GitHub personal access token, exported as `GITHUB_TOKEN` is required. [Setup](https://github.com/release-it/release-it#github-releases)

## 📜 Changelog

See the [CHANGELOG.md](./CHANGELOG.md) file. This file is auto-generated during the above mentioned release process.

## 🎁 Contributing

See the page titled "[Ways to Contribute](https://www.republicofocean.com/)" in the documentation.

## 🏛 License

```text
Copyright 2022 Republic of Ocean

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
