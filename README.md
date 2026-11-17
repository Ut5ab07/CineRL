# CineRL

CineRL is an educational movie-recommendation project that applies
reinforcement learning to the MovieLens 1M dataset. It includes:

- A data-preparation workflow for the raw MovieLens files.
- A tabular Q-learning experiment with a small, interpretable state/action
  space.
- A fitted Q-learning experiment using an `MLPRegressor`.
- A Flask web application that serves recommendations from the saved tabular
  Q-table.
- Research outputs, evaluation results, visualizations, and a project report.

The web application is **inference-only**. It loads the prepared catalog,
training ratings, and saved Q-table at startup; it does not retrain the agent
or modify the research artifacts.

## What the application does

The Discover page asks for:

1. A MovieLens user ID.
2. A preferred genre.
3. A satisfaction level: `Low`, `Medium`, or `High`.

The agent treats `(preferred genre, satisfaction)` as the current state and
selects the genre with the highest learned Q-value as its action. The
application then returns up to ten popular movies in that selected genre that
the user has not rated in the training split.

Recommendations are ordered using training-only popularity:

1. Highest rating count.
2. Highest average rating when rating counts are tied.

The interface also provides:

- **Discover**: generate movie recommendations.
- **Insights**: inspect the selected action and all Q-values for the latest
  session.
- **History**: review or clear locally stored recommendation sessions.
- **About**: view a short explanation of the reinforcement-learning loop.

## Project architecture

```text
Browser
  |
  v
Flask routes (web/routes.py)
  |-- HTML templates and static JavaScript/CSS
  |-- Recommender (web/recommender.py)
  |     |-- data/processed/movies.csv
  |     |-- data/processed/train_ratings.csv
  |     `-- data/processed/q_table.csv
  `-- HistoryStore (web/history.py)
        `-- data/recommendation_history.json
```

The application entry point is [app.py](app.py). It creates the Flask app,
loads the recommender, creates the history store, and registers the routes.

## Repository layout

```text
CineRL/
├── app.py                         Flask application entry point
├── requirements.txt               Python runtime dependencies
├── data/
│   ├── README                      MovieLens dataset documentation and license
│   ├── movies.dat                 Raw movie catalog
│   ├── ratings.dat               Raw user ratings
│   ├── users.dat                 Raw user metadata
│   ├── recommendation_history.json Local recommendation-session history
│   └── processed/                 Prepared data and model outputs
├── notebooks/
│   ├── 01_data_preparation.ipynb  Load, inspect, and split MovieLens data
│   ├── 02_q_learning.ipynb        Train and evaluate tabular Q-learning
│   └── 03_fitted_q_learning.ipynb Train and evaluate fitted Q-learning
├── web/
│   ├── history.py                 Thread-safe JSON history storage
│   ├── recommender.py             Q-table loading and inference
│   └── routes.py                  HTML and JSON API routes
├── templates/                     Flask/Jinja HTML pages
├── static/
│   ├── css/style.css              Application styles
│   └── js/script.js               API calls and page interactions
└── reports/
    ├── CineRL_Project_Report.pdf  Project report
    ├── CineRL_Project_Report.docx Editable project report
    └── figures/                   Generated charts
```

## Dataset

CineRL uses the MovieLens 1M dataset from the GroupLens Research Project:

- 1,000,209 anonymous ratings.
- Approximately 3,900 movies.
- 6,040 users.
- Ratings from 1 to 5 stars.
- 18 movie genres.

The raw files use the following formats:

```text
ratings.dat: UserID::MovieID::Rating::Timestamp
movies.dat:  MovieID::Title::Genres
users.dat:   UserID::Gender::Age::Occupation::Zip-code
```

The data-preparation notebook reads the files with `::` separators, preserves
the MovieLens movie titles and pipe-separated genres, and creates:

- `data/processed/movies.csv`
- `data/processed/train_ratings.csv`
- `data/processed/test_ratings.csv`

Ratings are split chronologically per user: the first 80% are used for
training and the remaining 20% for testing. This prevents future ratings from
being used to construct the training history.

See [data/README](data/README) for the original MovieLens usage conditions,
acknowledgement requirements, and citation information.

## Reinforcement-learning design

### State

The state is a pair:

```text
(preferred_genre, satisfaction)
```

There are 18 possible preferred genres and three satisfaction levels, giving
54 possible states.

### Actions

Each of the 18 MovieLens genres is an action:

```text
Action, Adventure, Animation, Children's, Comedy, Crime,
Documentary, Drama, Fantasy, Film-Noir, Horror, Musical,
Mystery, Romance, Sci-Fi, Thriller, War, Western
```

### Tabular Q-learning

The tabular experiment maintains a Q-table with one row per state and one
column per genre action. Its saved configuration is:

| Parameter | Value |
|---|---:|
| Random seed | 42 |
| Learning rate | 0.1 |
| Discount factor | 0.9 |
| Initial epsilon | 1.0 |
| Minimum epsilon | 0.05 |
| Epsilon decay | 0.9 |
| Epochs | 5 |
| Maximum steps per episode | 10 |
| Training users | 200 |

The resulting table is stored in
[data/processed/q_table.csv](data/processed/q_table.csv), with its parameters
recorded in [q_learning_config.json](data/processed/q_learning_config.json).

### Fitted Q-learning

The fitted experiment approximates Q-values with scikit-learn's
`MLPRegressor`. Its saved configuration uses:

- One hidden layer with 32 ReLU units.
- Discount factor (`gamma`) of 0.5.
- Learning rate of 0.001.
- A training sample of 50,000 transitions.
- Five maximum iterations.
- 39 total input features: 21 state features and 18 action features.
- 54 encoded states and 18 actions.

The fitted model and its outputs are kept under
[data/processed/fitted_q_learning](data/processed/fitted_q_learning). The
current Flask application uses the tabular Q-table, not the fitted model.

## Evaluation results

The saved evaluation compares both reinforcement-learning approaches with a
popularity baseline. Metrics are measured at ten recommendations:

| Model | Hit Rate@10 | Precision@10 | Recall@10 | Catalog Coverage |
|---|---:|---:|---:|---:|
| Q-learning | 0.3006 | 0.0527 | 0.0330 | 0.1028 |
| Popularity baseline | 0.3948 | 0.0810 | 0.0477 | 0.0301 |
| Fitted Q-learning | 0.1168 | 0.0185 | 0.0118 | 0.0306 |

These values are preserved in
[final_model_comparison.csv](data/processed/final_model_comparison.csv).
The results show the trade-off represented by this experiment: the popularity
baseline has the strongest measured accuracy, while tabular Q-learning
reaches substantially broader catalog coverage. The fitted model is included
for comparison and is not the model currently served by the application.

Additional training histories, Q-tables from exploration experiments,
recommendation exports, transition data, and charts are available under
[data/processed](data/processed) and [reports/figures](reports/figures).

## Installation

### Requirements

- Python 3.10 or newer is recommended.
- `pip`
- A terminal that can run a Flask development server.

Install the dependencies in a virtual environment:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

The runtime dependencies are:

- Flask 3.x
- NumPy 1.24 or newer
- pandas 2.0 or newer

## Run the web application

From the repository root:

```powershell
.venv\Scripts\Activate.ps1
python app.py
```

Open <http://127.0.0.1:5000> in a browser.

The server currently runs Flask in debug mode because this project is intended
for local research and demonstration. Do not use the development server as a
production deployment without adding production configuration, authentication,
logging, and a persistent database.

## API reference

### `GET /api/users`

Returns the user IDs available in the training ratings.

### `GET /api/genres`

Returns the 18 genre actions found in the saved Q-table.

### `POST /api/recommend`

Generates recommendations and records the session.

Request body:

```json
{
  "user_id": 1,
  "preferred_genre": "Action",
  "satisfaction": "High"
}
```

The response contains the input state, selected genre action, selected
Q-value, all genre alternatives ranked by Q-value, and the movie
recommendations:

```json
{
  "state": {
    "preferred_genre": "Action",
    "satisfaction": "High"
  },
  "selected_action": "Thriller",
  "selected_q_value": 4.0318,
  "alternatives": [],
  "recommendations": []
}
```

Invalid user IDs, genres, satisfaction levels, missing fields, or malformed
values return HTTP 400 with an `error` message.

### `GET /api/insights`

Returns the state, selected action, selected Q-value, and alternatives from the
most recent recommendation session. It returns HTTP 404 when no session exists.

### `GET /api/history`

Returns recommendation sessions in reverse chronological order.

### `DELETE /api/history`

Clears the local recommendation history and returns:

```json
{"status": "cleared"}
```

## Reproducing the research workflow

The notebooks are designed to be run from the `notebooks` directory in a
Jupyter environment:

1. Run `01_data_preparation.ipynb` to load the raw MovieLens files, inspect
   distributions, and create the processed train/test data.
2. Run `02_q_learning.ipynb` to train the tabular agent, evaluate it, and
   export the Q-table and training artifacts.
3. Run `03_fitted_q_learning.ipynb` to train the fitted Q-learning model and
   export its model, metrics, and recommendations.
4. Run `python app.py` from the repository root to use the saved tabular model
   in the web interface.

The notebooks use a fixed random seed of 42 for reproducible experiments.
Generated files in `data/processed` are the inputs expected by the current
web application, so rerunning notebooks should be done deliberately and
reviewed before replacing existing artifacts.

## Limitations and responsible use

- The application is a research/demo system, not a production recommender.
- The state representation is intentionally small and does not model detailed
  user preferences, context, recency, or demographic information.
- Recommendations use genre filtering and training-only popularity after the
  Q-learning action is selected; the Q-table does not score individual movies.
- Users and movies not represented in the processed training data cannot be
  selected by the current UI.
- Recommendation history is stored in a local JSON file and is capped at the
  latest 100 sessions.
- The history file is not a user-account system and should not be treated as
  secure or multi-user storage.
- The MovieLens data is provided for research purposes under the conditions in
  [data/README](data/README); it must not be redistributed or used
  commercially without the required permission.

## Citation and acknowledgements

If you publish results based on the MovieLens dataset, cite:

> F. Maxwell Harper and Joseph A. Konstan. 2015. The MovieLens Datasets:
> History and Context. ACM Transactions on Interactive Intelligent Systems
> (TiiS) 5, 4, Article 19 (December 2015), 19 pages.
> DOI: <https://doi.org/10.1145/2827872>

The dataset is provided by the GroupLens Research Project at the University of
Minnesota. Refer to [data/README](data/README) for the complete original
license, acknowledgement, and dataset notes.

## License

No separate software license is declared in this repository. The MovieLens
dataset has its own usage conditions, which are reproduced in
[data/README](data/README) and take precedence over any assumptions about
redistributing the bundled data.
