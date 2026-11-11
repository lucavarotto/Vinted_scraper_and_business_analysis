# Data-Driven Vinted Analytics

![R](https://img.shields.io/badge/R-randomForest%20%7C%20gbm%20%7C%20arules-276DC3?logo=r&logoColor=white)
![Python](https://img.shields.io/badge/Python-Selenium%20%7C%20pytrends-3776AB?logo=python&logoColor=white)
![Data](https://img.shields.io/badge/data-web%20scraping-09B1BA)
![Methods](https://img.shields.io/badge/methods-mixed%20effects%20%7C%20MBA%20%7C%20network-6E4AFF)

On Vinted anyone can list anything, which makes pricing a guess: ask too much
and the item sits there, ask too little and the seller loses money. This
project treats the platform as a client and answers four questions a consultant
would be handed — how much is an item worth, what makes a listing attractive,
what "good condition" means to the community, and which brands live in the same
wardrobe.

Everything rests on a dataset of **about 2,000 sneaker listings** scraped from
Vinted, enriched with **Google Trends** as a proxy for how much each brand is
searched for worldwide.

**Academic context.** Group project by Laura Martini, Gian Marco Tomietto and
Luca Varotto for *Strumenti Statistici per l'Analisi di Dati Aziendali*
(Statistical Tools for Business Data Analysis), M.Sc. in *Statistical Sciences*
at the University of Padua, academic year 2025/2026. The presentation is in
Italian, and so are the comments in the code.

---

## The pipeline

| Stage | File | What it does |
| --- | --- | --- |
| 1. Scraping | `Scrapers/Vinted_with_backup.py` | drives Selenium through the listings and saves one row per item, with a checkpoint system that survives a crash mid-run |
| | `Scrapers/Google_trends_globally.py` | queries Google Trends in batches for the average yearly interest of every brand, retrying on rate limits |
| 2. Cleaning | `Analisi/0_pulizia.R` | reads the raw CSV, folds the 282 colours into four classes and the free-text brands with regular expressions, sends the rare ones to *Altro*, log-transforms price and review counts, joins the Trends data and writes the modelling datasets |
| 3. Analysis | `Analisi/1_analisi.R` | the four pillars below, from the regressions to the network analysis |
| | `Analisi/utils.R` | custom estimation routines, including the mixed-effects trees and forests |

The two R scripts run with the working directory set to `Analisi/`: they read
the data from `../Dati/`, write the figures into `Plot/` and the fitted models
into `Modelli_stimati/`. `0_pulizia.R` pulls the raw data straight from this
repository over HTTPS, so it runs on a fresh clone.

---

## Repository structure

```
Vinted_scraper_and_business_analysis/
├── Scrapers/
│   ├── Vinted_with_backup.py        # the Vinted scraper
│   └── Google_trends_globally.py    # the Google Trends client
├── Analisi/
│   ├── 0_pulizia.R                  # cleaning and feature engineering
│   ├── 1_analisi.R                  # the four analytical pillars
│   ├── utils.R                      # mixed-effects trees and forests
│   ├── Plot/                        # the figures used in the presentation
│   └── Modelli_stimati/             # fitted models, kept local
├── Dati/
│   ├── dati.csv                     # the raw scraping output
│   ├── dati_puliti.Rdata            # the modelling datasets, from stage 2
│   ├── google_trends_global.csv     # brand interest, first batch
│   ├── google_trends_global_p2.csv  # brand interest, second batch
│   └── brand_pt2.csv                # the brands of the second batch
├── Martini_Tomietto_Varotto.pptx    # the final presentation (Italian)
├── .gitignore
└── README.md
```

The fitted models in `Analisi/Modelli_stimati/` are not published: they are
heavy and `1_analisi.R` rebuilds them.

---

## Results

### 1. Pricing recommendation

The target is the log price. Two effects dominate, both with a wide but
entirely positive confidence interval: a declared **item verification** is
worth about **+198%** and the **physical condition** of the shoe about
**+170%** [+121%, +229%].

Comparing the models on the test set, once the variables that leak the price
are removed:

| Model | R² | MAE (€) | RMSE (€) |
| --- | --- | --- | --- |
| Linear model on log(price) | 0.469 | 26.6 | 48.1 |
| Mixed model, random intercept by seller | 0.500 | 25.5 | 48.3 |
| Regression tree | — | 37.5 | 51.8 |
| Mixed-effects tree (MERT), brand and condition | 0.22 | 33.3 | 47.8 |
| Random forest | 0.23 | 29.2 | 47.5 |
| **Mixed-effects forest (MERF), brand** | **0.29** | **29.0** | **45.5** |
| Gradient boosting | 0.20 | 31.6 | 48.3 |

The hierarchical versions beat their flat counterparts every time: the brand is
a grouping level, not just another predictor. Even so, a mean absolute error of
€29 on a second-hand sneaker says the price is only partly explained by what a
listing declares.

### 2. What makes a listing attractive

The same exercise on the number of favourites, and the honest answer is that it
barely works:

| Model | R² | MAE | RMSE |
| --- | --- | --- | --- |
| Linear model on log1p | 0.113 | 12.1 | 21.8 |
| Negative binomial | 0.070 | 13.6 | 20.8 |
| Zero-inflated NB | 0.066 | 13.6 | 20.9 |
| Random forest | 0.16 | 13.1 | 19.4 |
| **Gradient boosting** | **0.18** | **12.8** | **19.2** |

The zero-inflated model collapses onto the plain negative binomial, and nothing
explains more than a fifth of the variance. Attention on a listing is driven by
something the structured fields do not capture.

### 3. What the community calls "good condition"

An ordinal probit on the declared quality, a strongly unbalanced response: 50%
of the items are *Ottime* and only 2% *Discrete*. Four strategies, in order:

| Strategy | Accuracy | Kappa | Ordinal MAE | Sensitivity on *Usato* |
| --- | --- | --- | --- | --- |
| Stepwise, five classes | 46% | 0.017 | 0.78 | collapses onto *Ottime* |
| Weighted, five classes | 32% | 0.134 | 1.04 | — |
| Three classes | 53% | 0.138 | 0.477 | 7.7% |
| **Three classes with weights** | **55%** | **0.207** | **0.32** | **66%** |

Accuracy moves by two points between the last two rows, while the sensitivity
on used items goes from 7.7% to 66%: the headline metric hides everything that
matters here. Stepwise selection drops nearly every seller-level variable,
which is the real finding — the community judges the product, not who is
selling it. The errors concentrate on the big brands, with a 61.6% error rate
on Nike.

### 4. Brand intelligence

**Market basket analysis** over the brands found in each wardrobe, minimum
support 2%, yields **66 association rules**. Nike and Adidas dominate the
support and act as hubs, while the lift rises sharply when the streetwear
ecosystem — Vans, Converse, Jordan, Puma — joins them, forming a dense cluster.

**Network analysis** on a symmetric brand-by-brand adjacency matrix over the
top 30 brands: Google Trends turns out **not** to be a significant covariate,
so the final model is a Social Relations Model without one. Its latent space
separates a fast-fashion neighbourhood, Zara, Shein, H&M, Bershka and
Stradivarius, from the streetwear one.

**Segmentation** of 1,912 products into four segments, with K-means on the
Euclidean distance and hierarchical clustering on Gower's: *Hype / Viral* (25%,
expensive and wanted), *Quiet Premium* (26%, expensive and in great condition
but ignored), and two low-price unsponsored groups. A multinomial model
profiling the segments finds **brand and sponsorship** to be the primary
drivers.

---

## Authors

**Laura Martini**, **Gian Marco Tomietto** and **Luca Varotto** — M.Sc. in
Statistical Sciences, University of Padua, academic year 2025/2026.
