# CineRL

CineRL is a Flask interface around the saved MovieLens Q-learning agent. It
loads `data/processed/q_table.csv` for inference and never retrains or
modifies the research artifacts.

## Run locally

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
python app.py
```

Open <http://127.0.0.1:5000>. Recommendation history is stored in
`data/recommendation_history.json` and is separate from the MovieLens datasets.

The Q-table state is `(preferred_genre, satisfaction)` and actions are the
18 MovieLens genres. Movie candidates are drawn from the catalog, exclude the
selected user's training-split history, and use training-only popularity for
stable ordering.
