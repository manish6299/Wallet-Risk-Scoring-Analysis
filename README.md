# Wallet-Risk-Scoring-Analysis


# Wallet Risk Scoring Analysis

This document explains the process of collecting data, selecting features, and developing a scoring method to assess the risk associated with Ethereum wallets.

## Data Collection

Details on how the wallet transaction data and Compound protocol interactions were collected using the Etherscan and The Graph APIs.

## Feature Selection

Rationale behind selecting the specific features used in the risk scoring model, including both directly obtained data and engineered features.

## Scoring Method

Explanation of the methodology used to calculate the risk scores for each wallet, including the scaling and weighting of features.

## Risk Indicator Justification

Justification for why each selected feature is considered a relevant indicator of risk for a wallet.

## Usage

Instructions on how to use the generated risk scores and potentially how to run the analysis script.

## Dependencies

List of necessary libraries and tools to run the analysis.
"""

### Data Collection Details

Data for each wallet was collected from two primary sources: the Etherscan API and The Graph API for Compound V2 and V3 subgraphs.

1.  **Etherscan API:**
    *   Standard transactions (`txlist`): Used to retrieve a history of Ether transactions for each wallet. From this, we extracted the total number of transactions (`num_txs`), the total amount of Ether sent out (`total_out_eth`), and the average Ether value per transaction (`avg_tx_value_eth`). The timestamps of these transactions were also used to estimate the wallet's age (`wallet_age_days`) and the total gas spent (`total_gas_spent`) over its lifetime.
    *   Token transfer events (`tokentx`): Used to retrieve a history of ERC-20 token transfers to the wallet. From this, we calculated the total amount of tokens received (`total_token_received`), normalizing for different token decimal places.

2.  **The Graph API (Compound V2 and V3 Subgraphs):**
    *   Compound V2 Subgraph: Queried to get details about a wallet's interactions with Compound V2 pools. This included the total supply balance (`compound_supply`), total borrow balance (`compound_borrow`), and the calculated collateral ratio (`compound_collateral_ratio = supply / borrow`).
    *   Compound V3 Subgraph: Queried to get details about a wallet's interactions with Compound V3 pools. This included the total collateral value in USD (`v3_collateral_usd`), total borrow value in USD (`v3_borrow_usd`), and the health score (`v3_health_score`), which indicates the proximity to liquidation.
    *   Liquidation Events: Queried from the Compound V2 subgraph to count the number of times a wallet has been liquidated (`liquidation_count`), a direct indicator of past risk events.

Each wallet ID was processed sequentially, fetching data from these APIs and combining the relevant metrics into a single record. A small time delay was included between requests to avoid hitting API rate limits.
"""

## Feature Selection

The selection of features for the risk scoring model was based on identifying metrics that indicate a wallet's activity, value transacted, engagement with DeFi protocols (specifically Compound), and past risk events. The process involved considering various data points available from the collected data and engineering new features where feasible.

Initially, a broad set of features were considered, including:
- Transaction count (`num_txs`)
- Total Ether sent out (`total_out_eth`)
- Average Ether value per transaction (`avg_tx_value_eth`)
- Total tokens received (`total_token_received`)
- Compound V2 supply, borrow, and collateral ratio (`compound_supply`, `compound_borrow`, `compound_collateral_ratio`)
- Compound V3 collateral, borrow, and health score (`v3_collateral_usd`, `v3_borrow_usd`, `v3_health_score`)
- Liquidation count (`liquidation_count`)

New features engineered from the transaction data included:
- Wallet age in days (`wallet_age_days`): Calculated from the timestamp difference between the first and last transaction.
- Total gas spent (`total_gas_spent`): Sum of gas used multiplied by gas price for all transactions, converted to Ether.

The final selected features for the model are:
- num_txs: Included as it has a high correlation with the current score and indicates overall activity level.
- total_out_eth: Included due to high correlation with the score, reflecting the value transacted out.
- avg_tx_value_eth: Included as it shows high correlation with the score and represents typical transaction size.
- total_token_received: Included as it is conceptually relevant to wallet activity and participation in token ecosystems, despite a lower current correlation.
- wallet_age_days: Included to provide context on the wallet's longevity and potential stability.
- total_gas_spent: Included as a proxy for interaction complexity and frequency, particularly with smart contracts.
- compound_supply, compound_borrow, compound_collateral_ratio: Included as they are critical for assessing risk within Compound V2, even if they had zero variance in the initial dataset. They are essential if wallets use V2.
- v3_collateral_usd, v3_borrow_usd, v3_health_score: Included as they are critical for assessing risk within Compound V3, even if they had zero variance in the initial dataset. They are essential if wallets use V3.
- liquidation_count: Included as a direct indicator of past risk events, even if it had zero variance in the initial dataset. It is highly relevant if any liquidation events exist.

Features that were considered but excluded in this iteration due to complexity or data limitations include:
- Interaction with Specific Smart Contracts (beyond Compound): Requires external lists of contract addresses and more complex filtering logic.
- Incoming Transactions and Sources/Outbound Transaction Destinations: Requires sophisticated graph analysis or external data sources to map entities and assess risk, which is beyond the scope of this iteration.
- Specific Token Holdings/Balances: Obtaining accurate, real-time token balances for a wide range of tokens via API is challenging. total_token_received is used as a reasonable starting point.

This selection aims to balance direct indicators of activity and value with protocol-specific risk metrics and temporal context, while remaining feasible given the data sources.
"""



## Scoring Method

The risk score for each wallet is calculated based on a weighted sum of several key features. The process involves normalizing the feature values using `MinMaxScaler`, applying specific weights to each feature based on its perceived impact on risk, and then scaling the final score to a range of 0 to 1000.

1.  **MinMaxScaler:** The `MinMaxScaler` is used to scale each selected feature's values to a range between 0 and 1. This is crucial because the raw values of different features (e.g., number of transactions vs. total Ether sent) can vary significantly in magnitude. Scaling ensures that each feature contributes to the final score based on its relative value within the dataset, rather than its absolute size. The formula for MinMaxScaler is:

    $$X_{scaled} = \\frac{X - X_{min}}{X_{max} - X_{min}}$$

2.  **Feature Weighting:** Each scaled feature is multiplied by a predefined weight. These weights reflect the importance assigned to each feature in indicating risk. Features where a higher value is associated with *higher* risk (e.g., `num_txs`, `total_out_eth`, `compound_borrow`, `v3_borrow_usd`, `liquidation_count`, `total_gas_spent`) are weighted directly. Features where a higher value is associated with *lower* risk (e.g., `compound_supply`, `compound_collateral_ratio`, `v3_collateral_usd`, `v3_health_score`, `wallet_age_days`) are weighted using `(1 - scaled_value)` so that a higher original value results in a lower contribution to the final risk score.

3.  **Raw Score Calculation:** A raw score is calculated for each wallet by summing the weighted scaled features:

    $$Raw Score = \\sum_{i=1}^{n} Weight_i \\times Scaled\_Feature_i$$

    where $Scaled\_Feature_i$ is the scaled value of the feature $i$, and for features indicating lower risk with higher values, $Scaled\_Feature_i$ becomes $(1 - Scaled\_Feature_i)$ before weighting.

4.  **Final Score Scaling:** The raw scores, which are initially in an arbitrary range, are then scaled to a standardized range of 0 to 1000. This is done by dividing each raw score by the maximum raw score across all wallets and multiplying by 1000:

    $$Final Score = \\frac{Raw Score}{Max(Raw Scores)} \\times 1000$$

5.  **Rounding:** The final scaled scores are rounded to the nearest integer to provide a simple, discrete risk score between 0 and 1000.

This method provides a transparent and adjustable way to quantify the risk associated with each wallet based on a combination of their on-chain activity and DeFi interactions.
"""

## Risk Indicator Justification

This section explains why each of the selected features is considered a relevant indicator of risk for a wallet in the context of decentralized finance, particularly lending protocols like Compound.

-   **num_txs (Number of Transactions):** A higher number of transactions generally indicates a more active wallet. While high activity can be a sign of legitimate use, an unusually high frequency or volume of transactions could also be associated with automated trading bots or potentially suspicious activity. For risk scoring, higher transaction count is often considered to contribute to a higher risk score as it increases the surface area for potential issues or complex interactions.
-   **total_out_eth (Total Ether Sent Out):** This metric represents the cumulative value of Ether transferred from the wallet. Large outflows could indicate significant trading, transfers to other addresses (which might be associated with other entities), or participation in various protocols. A higher total outflow is generally associated with a more active and potentially riskier profile, as it implies larger value movements that could be subject to market volatility or protocol-specific risks.
-   **avg_tx_value_eth (Average Transaction Value in Ether):** This feature provides insight into the typical size of Ether transactions originating from the wallet. Wallets with a higher average transaction value might be involved in larger-scale operations or investments. While not a direct risk in itself, it can amplify the impact of other risk factors; for example, a large average transaction value combined with high borrowing could indicate significant exposure. Higher average transaction value is considered to contribute to a higher risk score.
-   **total_token_received (Total Tokens Received):** This represents the cumulative amount of various ERC-20 tokens transferred into the wallet. High token inflows can indicate participation in airdrops, receipt of protocol rewards, or large-scale token accumulation. While receiving tokens isn't inherently risky, a very high volume of diverse token receipts might suggest engagement in speculative activities or interactions with numerous potentially unaudited contracts, which could increase risk. Higher total tokens received are considered to contribute to a higher risk score.
-   **compound_supply (Compound V2 Supply Balance Underlying):** This is the total value of assets supplied by the wallet to Compound V2 lending pools. A higher supply balance generally indicates a user providing liquidity, which is a core function of the protocol. A higher supply balance is typically associated with *lower* risk from the perspective of the wallet's health within Compound V2, as it contributes to their collateral.
-   **compound_borrow (Compound V2 Borrow Balance Underlying):** This is the total value of assets borrowed by the wallet from Compound V2 lending pools. Borrowing introduces liquidation risk. A higher borrow balance, especially relative to supplied collateral, increases the likelihood of being liquidated if the value of collateral drops or the value of borrowed assets rises. A higher borrow balance is a direct indicator of *higher* risk.
-   **compound_collateral_ratio (Compound V2 Collateral Ratio):** Calculated as Supply / Borrow. This ratio is a critical health metric in Compound V2. A higher collateral ratio indicates a larger buffer between the current state and the liquidation threshold, meaning lower risk. A ratio of 1 or less for a borrowing position indicates the wallet is at or below the liquidation point. A higher collateral ratio is associated with *lower* risk.
-   **v3_collateral_usd (Compound V3 Total Collateral Value USD):** This is the total USD value of assets supplied as collateral to Compound V3 markets. Similar to Compound V2 supply, higher collateral value in V3 contributes to the wallet's health and ability to borrow safely. A higher collateral value is associated with *lower* risk within Compound V3.
-   **v3_borrow_usd (Compound V3 Total Borrow Value USD):** This is the total USD value of assets borrowed from Compound V3 markets. Similar to Compound V2 borrow, higher borrow value in V3 increases liquidation risk. A higher borrow value is a direct indicator of *higher* risk within Compound V3.
-   **v3_health_score (Compound V3 Health Score):** This is a metric calculated by Compound V3 indicating the health of a wallet's borrowing position. A higher health score means the position is safer and further away from liquidation. A health score of 1 indicates the position is at the liquidation threshold. A higher health score is associated with *lower* risk.
-   **liquidation_count (Compound V2 Liquidation Events Count):** This is a direct count of how many times the wallet's borrowing position has been liquidated in Compound V2. Past liquidations are strong indicators of previous risk-taking behavior or inability to manage positions effectively. A higher liquidation count is associated with *higher* risk.
-   **wallet_age_days (Wallet Age in Days):** The duration since the wallet's first transaction. An older wallet might suggest a more established user with potentially more experience navigating the ecosystem. Newer wallets might be associated with more speculative or short-term activity. While not a definitive risk indicator, older wallets are sometimes considered to have slightly *lower* risk due to longevity, assuming consistent activity.
-   **total_gas_spent (Total Gas Spent):** The cumulative amount of Ether spent on transaction fees. Higher gas expenditure can indicate more frequent or complex interactions, especially with smart contracts. While high gas spending can be a sign of legitimate heavy usage, it can also be associated with complex, potentially risky strategies or interactions with inefficient/costly protocols. Higher total gas spent is considered to contribute to a *higher* risk score.
"""


## Usage

To replicate this analysis and generate risk scores for a list of Ethereum wallets, follow these steps:

1.  **Environment Setup:** Ensure you have Python installed. Install the necessary libraries using pip:
    ```bash
    pip install pandas requests numpy scikit-learn matplotlib
    ```

2.  **API Keys:** Obtain API keys for Etherscan and The Graph (though The Graph is often publicly accessible, Etherscan requires a key). Replace the placeholder API keys in the `CONFIGURATION` section of the notebook with your actual keys.

3.  **Wallet List:** Prepare a CSV file named `Wallet id - Sheet1.csv` containing a single column named `wallet_id` with the list of Ethereum wallet addresses you want to analyze. Place this file in the same directory as the notebook.

4.  **Run Notebook:** Execute the notebook cells sequentially from top to bottom.
    *   The initial cells define necessary functions and configurations.
    *   The data collection loop will fetch data for each wallet (this may take some time depending on the number of wallets and API rate limits). Progress will be printed to the console.
    *   The scoring cells will calculate the risk scores based on the collected data and defined weights.
    *   The final cells will display summary statistics, correlations, and a histogram of the scores, and save the results to CSV files.

Upon successful execution, the risk scores will be available in the DataFrame and saved to `wallet_risk_scores_improved.csv`.



## Dependencies

To run this analysis, you will need the following Python libraries installed:

*   `pandas`: For data manipulation and analysis.
*   `requests`: For making API calls to Etherscan and The Graph.
*   `numpy`: For numerical operations, particularly for use with scikit-learn.
*   `scikit-learn`: For data scaling using `MinMaxScaler`.
*   `matplotlib`: For generating plots, such as the histogram of scores.

