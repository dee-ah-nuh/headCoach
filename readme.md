## Real Madrid Match Prediction Pipeline

🎯 Problem Statement
This project builds a production-grade machine learning pipeline to predict Real Madrid match outcomes (Win/Draw/Loss) in La Liga by leveraging historical match data and advanced performance metrics. The challenge lies in accurately forecasting soccer results—a notoriously unpredictable domain—by combining traditional statistical features (goals scored/conceded, home/away performance) with expected goals (xG) data that captures underlying team quality beyond final scores.
We tackle this by implementing an end-to-end data engineering solution: Python scrapers extract match-level data from FBref and land it as JSON in S3; PySpark jobs on AWS EMR transform raw data into partitioned Parquet files optimized for analytics; Snowflake serves as the data warehouse where dbt models engineer features like rolling averages, rest days, and opponent strength indicators; finally, a classification model (Multinomial Logistic Regression or XGBoost) trained on 3 seasons of historical data (~400 matches) generates predictions with confidence probabilities 24-48 hours before each match.
The entire pipeline is orchestrated via Airflow with idempotent design, automated validation comparing predictions against actual results, and infrastructure-as-code via Terraform—demonstrating modern data lake architecture, distributed processing with Spark, and scalable ML operationalization patterns.


#### Problem Considerations
---

- **Late Data StrategyProblem:** FBref may not publish match results immediately after games end. Match times can be delayed. Opponent data may be missing.Our Approach:Watermarking vs Reprocessing Windows

- **Problem:** FBref might add new stats (e.g., "progressive passes"), change field names, or restructure HTML.

- **Problem:** Bad data corrupts predictions. How do we catch issues before they break downstream consumers?


### Architecture
---
```
erDiagram
    %% ===========================================
    %% REAL MADRID MATCH PREDICTION - DATA MODEL
    %% ===========================================

    TEAMS {
        string team_id PK "FBref team ID (e.g., 53a2f082)"
        string team_name "Official team name"
        string team_name_short "Short name"
        string season "Season participated"
        timestamp scraped_at "When scraped"
    }

    TEAM_SEASON_STATS {
        string team_id PK,FK "Links to TEAMS"
        string season PK "Season (e.g., 2024-25)"
        int matches_played "MP"
        int wins "W"
        int draws "D"
        int losses "L"
        int goals_for "GF"
        int goals_against "GA"
        int goal_difference "GD"
        int points "Pts"
        float points_per_match "Pts/MP"
        float xg "Expected Goals"
        float xga "Expected Goals Against"
        float xgd "xG Difference"
        float xgd_per_90 "xGD/90"
        int goals "Total goals scored"
        int assists "Total assists"
        float npxg "Non-Penalty xG"
        int progressive_carries "PrgC"
        int progressive_passes "PrgP"
        int shots "Total shots"
        int shots_on_target "SoT"
        float shot_on_target_pct "SoT%"
        float goals_per_shot "G/Sh"
        float xg_per_shot "xG/Sh"
        int tackles "Tkl"
        int tackles_won "TklW"
        int interceptions "Int"
        int blocks "Blocks"
        int clearances "Clr"
        int errors "Err"
        int yellow_cards "CrdY"
        int red_cards "CrdR"
        int fouls "Fls"
        int aerials_won "Won"
        float aerials_won_pct "Won%"
        timestamp scraped_at "When scraped"
    }

    MATCHES {
        string match_id PK "Unique match identifier"
        date date "Match date"
        string season FK "Links to season"
        string competition "La Liga"
        int matchweek "Round number"
        string home_team FK "Links to TEAMS"
        string away_team FK "Links to TEAMS"
        string venue "Stadium name"
        string status "SCHEDULED/COMPLETED"
        int home_goals "Goals by home team"
        int away_goals "Goals by away team"
        string winner "HOME/AWAY/DRAW"
        string real_madrid_result "WIN/DRAW/LOSS (target variable)"
        float home_xg "Home expected goals"
        float away_xg "Away expected goals"
        int home_possession "Home possession %"
        int away_possession "Away possession %"
        int home_shots "Home total shots"
        int away_shots "Away total shots"
        int home_shots_on_target "Home SoT"
        int away_shots_on_target "Away SoT"
        int home_passes_completed "Home passes"
        int away_passes_completed "Away passes"
        float home_pass_completion_pct "Home pass %"
        float away_pass_completion_pct "Away pass %"
        int home_corners "Home corners"
        int away_corners "Away corners"
        int home_fouls "Home fouls"
        int away_fouls "Away fouls"
        int home_yellow_cards "Home yellows"
        int away_yellow_cards "Away yellows"
        int home_red_cards "Home reds"
        int away_red_cards "Away reds"
        int attendance "Match attendance"
        string source_url "FBref URL"
        timestamp scraped_at "When scraped"
    }

    PREDICTIONS {
        string prediction_id PK "Unique prediction ID"
        string match_id FK "Links to MATCHES"
        string predicted_outcome "WIN/DRAW/LOSS"
        float confidence_win "P(Real Madrid Win)"
        float confidence_draw "P(Draw)"
        float confidence_loss "P(Real Madrid Loss)"
        string model_version "Model identifier"
        json features_used "Feature snapshot"
        timestamp predicted_at "When prediction made"
    }

    PREDICTION_VALIDATION {
        string validation_id PK "Unique validation ID"
        string match_id FK "Links to MATCHES"
        string prediction_id FK "Links to PREDICTIONS"
        string predicted_outcome "What model predicted"
        string actual_outcome "What actually happened"
        boolean was_correct "Did prediction match?"
        float confidence_of_prediction "Confidence of predicted outcome"
        timestamp validated_at "When validated"
    }

    %% ===========================================
    %% RELATIONSHIPS
    %% ===========================================

    TEAMS ||--o{ TEAM_SEASON_STATS : "has stats per season"
    TEAMS ||--o{ MATCHES : "plays as home_team"
    TEAMS ||--o{ MATCHES : "plays as away_team"
    MATCHES ||--o| PREDICTIONS : "has prediction"
    MATCHES ||--o| PREDICTION_VALIDATION : "has validation"
    PREDICTIONS ||--|| PREDICTION_VALIDATION : "validated by"
    ```
