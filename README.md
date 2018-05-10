<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [Server Documentation](#server-documentation)
    - [Username and password for authentication](#username-and-password-for-authentication)
    - [Required values from experiment submissions](#required-values-from-experiment-submissions)
    - [Retrieving experiment results](#retrieving-experiment-results)
    - [Dynamic experiment result retrieval as JSON](#dynamic-experiment-result-retrieval-as-json)
    - [Deploying the Server](#deploying-the-server)
        - [Deployment with Heroku](#deployment-with-heroku)
        - [Local (Offline) Deployment](#local-offline-deployment)
            - [First-time installation (requires internet connection)](#first-time-installation-requires-internet-connection)
            - [Deployment](#deployment)
- [Experiments (Frontend)](#experiments-frontend)
- [Additional Notes](#additional-notes)

<!-- markdown-toc end -->

This is a server backend to run simple psychological experiments in the browser and online. It
helps receive, store and retrieve data. Work on this project was funded via the project
[Pro^3](http://www.xprag.de/?page_id=4759), which is part of the [XPRAG.de](http://www.xprag.de/) funded by the German Research
Foundation (DFG Schwerpunktprogramm 1727).

If you encounter any bugs during your experiments please [submit an issue](https://github.com/x-ji/ProComPrag/issues).

A live version of the server is currently deployed at https://procomprag.herokuapp.com

For the documentation of the entire _babe project, please refer to the [project site](https://b-a-b-e.github.io/babe_site)

# Server Documentation
This section documents the server program.

## Username and password for authentication
The app now comes with [Basic access Authentication](https://en.wikipedia.org/wiki/Basic_access_authentication). The username and password are set under `config :procomprag, :authentication`.

For local deployment (`:dev` environment), the default username is `default` and the default password is `password`. You may change it in `dev.exs`.

If you're deploying on Heroku, be sure to set environment variables `AUTH_USERNAME` and `AUTH_PASSWORD`, either via Heroku command line tool or in the Heroku user interface, under `Settings` tab.
## Required values from experiment submissions

The server expects to receive results from experiments which are structured in a particular
way, usually stored in variable `exp.data`. For a minimal example, look at the
[Minimal Template](https://github.com/ProComPrag/MinimalTemplate), together with the
documentation of the front end. Data is submitted to the server via HTTP POST.

Data in `exp.data` requires **four crucial values** for the data to be processable by the
server:
- `author`: The author of this experiment
- `experiment_id`: The identifier (can be a string) that the author uses to name this experiment
- `trials`: A JSON array containing trial objects. The keys contained in each trial object should be the same.

Additionally, an optional array named `trial_keys_order`, which specifies the order in which the trial data should be
 printed in the CSV output, can be included. If this array is not included, the trial data will be printed in alphabetical order, which might not be ideal.

Any additional keys in the `exp.data` object will be printed in the CSV file output as well. However, note that the
basis for CSV generation will be the `trials` array, i.e. each trial within the `trials` array will result in one row in the final CSV file.

An example object from the [Minimal Template](https://github.com/ProComPrag/MinimalTemplate) is shown below:
```json
{
   "author":"Random Jane",
   "experiment_id":"MinimalTemplate",
   "description":"A minimal template for a browser-based experiment which can be deployed in several ways",
   "startDateTime":"Sun Apr 01 2018 21:31:56 GMT+0200 (CEST)",
   "total_exp_time_minutes":0.1122,
   "trial_keys_order": [
     "trial_type",
     "trial_number",
     "RT",
     "age",
     "option1",
     "option2",
     "response",
     "languages",
     "education",
     "gender",
     "comments"
   ],
   "trial_data":[
      {
         "trial_type":"practice",
         "trial_number":1,
         "question":"Where is your head?",
         "option1":"here",
         "option2":"there",
         "response":"here",
         "RT":898,
         "age":"",
         "gender":"",
         "education":"",
         "languages":"",
         "comments":""
      },
      {
         "trial_type":"practice",
         "trial_number":2,
         "question":"What's on the bread?",
         "option1":"jam",
         "option2":"ham",
         "response":"jam",
         "RT":1048,
         "age":"",
         "gender":"",
         "education":"",
         "languages":"",
         "comments":""
      },
      {
         "trial_type":"main",
         "trial_number":1,
         "question":"How are you today?",
         "option1":"fine",
         "option2":"great",
         "response":"fine",
         "RT":724,
         "age":"",
         "gender":"",
         "education":"",
         "languages":"",
         "comments":""
      },
      {
         "trial_type":"main",
         "trial_number":2,
         "question":"What is the weather like?",
         "option1":"shiny",
         "option2":"rainbow",
         "response":"shiny",
         "RT":384,
         "age":"",
         "gender":"",
         "education":"",
         "languages":"",
         "comments":""
      }
   ]
}
```

When an experiment is finished, instead of sending it with `mmturkey` to the interface provided by MTurk/using the
original `turk.submit(exp.data)`, please POST the JSON to the following web address: `{SERVER_ADDRESS}/api/submit_experiment`, e.g. https://procomprag.herokuapp.com/api/submit_experiment.

Note that to [POST a JSON object correctly](https://stackoverflow.com/questions/12693947/jquery-ajax-how-to-send-json-instead-of-querystring),
 one needs to specify the `Content-Type` header as `application/json`, and use `JSON.stringify` to encode the data first.

The following is an example for the `POST` call using jQuery.

```javascript
$.ajax({
  type: 'POST',
  url: 'https://procomprag.herokuapp.com/api/submit_experiment',
  // url: 'http://localhost:4000/api/submit_experiment',
  crossDomain: true,
	contentType: 'application/json',
	data: JSON.stringify(exp.data),
  success: function(responseData, textStatus, jqXHR) {
    console.log(textStatus)
  },
  error: function(responseData,textStatus, errorThrown) {
    alert('Submission failed.');
  }
})
```

The reason for error would most likely be missing mandatory fields (i.e. `author`, `experiment_id`, `description`, `trials`) in the JSON file.

Note that `crossDomain: true` is needed since the server domain will likely be different the domain where the experiment is presented to the participant.

## Retrieving experiment results
Just visit the server (e.g. at https://procomprag.herokuapp.com), enter the `experiment_id` and `author` originally contained within the JSON file, and hit "Submit". Authentication mechanisms might be added later, if necessary.

## Dynamic experiment result retrieval as JSON
For some experiments, there might be a need to fetch and use the data from previous experiment submissions in order to dynamically adjust the future assignments.

One can now specify the keys that should be fetched in the "Edit Experiment" user interface. Then, with a HTTP GET call to the `dynamic_retrieval` endpoint, specifying `author` and `experiment_id`, e.g. https://procomprag.herokuapps.com/api/dynamic_retrieval?author=RandomJane&experiment_id=MinimalTemplateDEMO, one will be able to get a JSON object that contains the results so far.

Example response after specifying `option_chosen`, `RT` and `timeSpent` as keys on the `MinimalTemplate` experiment:

```json
[
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 506
            },
            {
                "option_chosen": "jam",
                "RT": 110
            },
            {
                "option_chosen": "fine",
                "RT": 573
            },
            {
                "option_chosen": "jam",
                "RT": 122
            }
        ],
        "timeSpent": 0.07485
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 878
            },
            {
                "option_chosen": "jam",
                "RT": 108
            },
            {
                "option_chosen": "jam",
                "RT": 519
            },
            {
                "option_chosen": "here",
                "RT": 131
            }
        ],
        "timeSpent": 0.19921666666666665
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 502
            },
            {
                "option_chosen": "jam",
                "RT": 223
            },
            {
                "option_chosen": "here",
                "RT": 711
            },
            {
                "option_chosen": "shiny",
                "RT": 109
            }
        ],
        "timeSpent": 0.07365
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 446
            },
            {
                "option_chosen": "jam",
                "RT": 325
            },
            {
                "option_chosen": "here",
                "RT": 457
            },
            {
                "option_chosen": "shiny",
                "RT": 130
            }
        ],
        "timeSpent": 0.0691
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 566
            },
            {
                "option_chosen": "jam",
                "RT": 119
            },
            {
                "option_chosen": "here",
                "RT": 489
            },
            {
                "option_chosen": "fine",
                "RT": 104
            }
        ],
        "timeSpent": 0.08233333333333333
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 566
            },
            {
                "option_chosen": "jam",
                "RT": 119
            },
            {
                "option_chosen": "here",
                "RT": 489
            },
            {
                "option_chosen": "fine",
                "RT": 104
            }
        ],
        "timeSpent": 0.08233333333333333
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 566
            },
            {
                "option_chosen": "jam",
                "RT": 119
            },
            {
                "option_chosen": "here",
                "RT": 489
            },
            {
                "option_chosen": "fine",
                "RT": 104
            }
        ],
        "timeSpent": 0.08233333333333333
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 566
            },
            {
                "option_chosen": "jam",
                "RT": 119
            },
            {
                "option_chosen": "here",
                "RT": 489
            },
            {
                "option_chosen": "fine",
                "RT": 104
            }
        ],
        "timeSpent": 0.08233333333333333
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 566
            },
            {
                "option_chosen": "jam",
                "RT": 119
            },
            {
                "option_chosen": "here",
                "RT": 489
            },
            {
                "option_chosen": "fine",
                "RT": 104
            }
        ],
        "timeSpent": 0.08233333333333333
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 566
            },
            {
                "option_chosen": "jam",
                "RT": 119
            },
            {
                "option_chosen": "here",
                "RT": 489
            },
            {
                "option_chosen": "fine",
                "RT": 104
            }
        ],
        "timeSpent": 0.08233333333333333
    },
    {
        "trials": [
            {
                "option_chosen": "here",
                "RT": 566
            },
            {
                "option_chosen": "jam",
                "RT": 119
            },
            {
                "option_chosen": "here",
                "RT": 489
            },
            {
                "option_chosen": "fine",
                "RT": 104
            }
        ],
        "timeSpent": 0.08233333333333333
    }
]
```
## Deploying the Server
This section documents some methods one can use to deploy the server, for both online and offline usages.

### Deployment with Heroku
[Heroku](https://www.heroku.com/) makes it easy to deploy an web app without having to manually manage the infrastructure. It has a free starter tier, which should be sufficient for the purpose of running experiments.

There is an [official guide](https://hexdocs.pm/phoenix/heroku.html) from Phoenix framework on deploying on Heroku. The deployment procedure is based on this guide, but differs in some places.

0. Ensure that you have [the Phoenix Framework installed](https://hexdocs.pm/phoenix/installation.html) and working. However, if you just want to deploy this server and do no development work/change on it at all, you may skip this step.

1. Ensure that you have a [Heroku account](https://signup.heroku.com/) already, and have the [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli) installed and working on your computer.

2. Ensure you have [Git](https://git-scm.com/downloads) installed. Clone this git repo with `git clone https://github.com/ProComPrag/ProComPrag.git` or `git clone git@github.com:ProComPrag/ProComPrag.git`.

3. `cd` into the project directory just cloned from your Terminal (or cmd.exe on Windows).

4. Run `heroku create --buildpack "https://github.com/HashNuke/heroku-buildpack-elixir.git"`

5. Run `heroku buildpacks:add https://github.com/gjaldon/heroku-buildpack-phoenix-static.git`

  (N.B.: Although the command line output tells you to run `git push heroku master`, don't do it yet.)

6. You may want to change the application name instead of using the default name. In that case, go to the Heroku Dashboard, find the newly created app, and edit the name in `Settings` panel.

7. Edit line 17 of the file `config/prod.exs`. Replace the part `procomprag.herokuapp.com` after `host` with the app name (shown when you first ran `heroku create`, e.g. `mysterious-meadow-6277.herokuapp.com`, or the app name that you set at step 6, e.g.  `appname.herokuapp.com`). You shouldn't need to modify anything else.

8. Ensure that you're at the top-level project directory. Run
```
heroku addons:create heroku-postgresql:hobby-dev
heroku config:set POOL_SIZE=18
```

9. Run `mix phx.gen.secret`. Then run `heroku config:set SECRET_KEY_BASE="OUTPUT"`, where `OUTPUT` should be the output of the `mix phx.gen.secret` step.

  Note: If you don't have Phoenix framework installed on your computer, you may choose to use some other random generator for this task, which essentially asks for a random 64-character secret. On Mac and Linux, you may run `openssl rand -base64 64`. Or you may use an online password generator [such as the one offered by LastPass](https://lastpass.com/generatepassword.php).

10. Run `git add config/prod.exs`, then `git commit -m "Set app URL"`.

11. Run `git push heroku master`. This will push the repo to the git remote at Heroku (instead of the original remote at Github), and deploy the app.

12. Run `heroku run "POOL_SIZE=2 mix ecto.migrate"`

13. Now, `heroku open` should open the frontpage of the app.

### Local (Offline) Deployment
Normally, running the server in a local development environment would involve installing and configuring Elixir and PostgreSQL. To simplify the development flow, [Docker](https://www.docker.com/) is used instead.

#### First-time installation (requires internet connection)

The following steps require an internet connection. After they are finished, the server can be launched offline.

1. Install Docker from https://docs.docker.com/install/. You may have to launch the application once in order to let it install its command line tools. Ensure that it's running by typing `docker version` in a terminal (e.g., the Terminal app on MacOS or cmd.exe on Windows).

  Note:
  - Although the Docker app on Windows and Mac asks for login credentials to Docker Hub, they are not needed for local deployment . You can proceed without creating any Docker account/logging in.
  - Linux users would need to install `docker-compose` separately. See relevant instructions at https://docs.docker.com/compose/install/.

2. Ensure you have [Git](https://git-scm.com/downloads) installed. Clone the server repo with `git clone https://github.com/b-a-b-e/ProComPrag.git` or `git clone git@github.com:ProComPrag/ProComPrag.git`.

3. Open a terminal (e.g., the Terminal app on MacOS or cmd.exe on Windows), `cd` into the project directory just cloned via git.

4. For the first-time setup, run in the terminal
  ```
  docker volume create --name procomprag-app-volume -d local
  docker volume create --name procomprag-db-volume -d local
  docker-compose run --rm web bash -c "mix deps.get && npm install && node node_modules/brunch/bin/brunch build && mix ecto.migrate"
  ```

#### Deployment

After first-time installation, you can launch a local server instance which sets up the experiment in your browser and stores the results.

1. Run `docker-compose up` to launch the application every time you want to run the server. Wait until the line `web_1  | [info] Running ProComPrag.Endpoint with Cowboy using http://0.0.0.0:4000` appears in the terminal.

2. Visit `localhost:4000` in your browser. You should see the server up and running.

  Note: Windows 7 users who installed *Docker Machine* might need to find out the IP address used by `docker-machine` instead of `localhost`. See [Docker documentation](https://docs.docker.com/get-started/part2/#build-the-app) for details.

3. Use <kbd>Ctrl + C</kbd> to shut down the server.

Note that the database for storing experiment results is stored at `/var/lib/docker/volumes/procomprag-volume/_data` folder by default. As long as this folder is preserved, experiment results should persist as well.


# Experiments (Frontend)
This program is intended to serve as the backend which stores and returns experiment results. An experiment frontend is normally written as a set of static webpages to be hosted on a hosting provider (e.g. [Github Pages](https://pages.github.com/)) and loaded in the participant's browser.

For detailed documentation on the structure and deployment of experiments, please refer to the [minimal template](https://github.com/b-a-b-e/MinimalTemplate/) and the [_babe documentation](https://b-a-b-e.github.io/babe_site/).

# Additional Notes
- The assumption on the server side when receiving the experiments is that each set of experiment results would have the same keys in the JSON file submitted and that each trial n an experiment would have the same keys in an object named `trials`. Violating this assumption might lead to failure in the CSV generation process. Look at `norming.js` files in the sample experiments for details.

  If new experiments are created by modifying the existing experiment examples, they should work fine.

- Please avoid using arrays to store the experiment results as much as possible. For example, if the participant is allowed to enter three answers on one trial, it would be better to store the three answers under three separate keys, instead of an array under one key.

  However, if an array is used regardless, its items will be separated by a `|` (pipe) in the retrieved CSV file.

- There is limited guarantee on DB security on Heroku's Hobby grade. The experiment authors are expected to take responsibility of the results. They should retrieve them and perform backups as soon as possible.

- This app is based on Phoenix Framework and written in Elixir. If you wish to modify the app, please look at the resources available at:
  - Official website: http://www.phoenixframework.org/
  - Guides: http://phoenixframework.org/docs/overview
  - Docs: https://hexdocs.pm/phoenix
  - Mailing list: http://groups.google.com/group/phoenix-talk
  - Source: https://github.com/phoenixframework/phoenix
