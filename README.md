# XGBoost-Mini
This advanced library implements a fully functional, optimized, and native XGBoost (Extreme Gradient Boosting) binary classification model, allowing you to train an ensemble of decision trees and perform real-time inference directly on price data and technical indicators on TradingView.

# 🔷 1. User-Defined Types (UDTs)
The code leverages Pine Script v6 data structures to define the model architecture:
XGBTreeDepth3: Represents a single weak learner with a fixed depth of 3 levels. It stores feature indices, split thresholds, information gains for each node, and the terminal leaf weights (w0 through w7) for all 8 possible leaf regions.
XGBModel: Encapsulates the entire trained tree ensemble, the best recorded validation loss (best_val_loss), and the optimal number of trees to retain (best_tree_count).
SplitCandidate: An internal helper structure used to evaluate optimal split points during tree growth.

# 🔷 2. Inference & Analysis Methods
predict_tree: Traverses the depth-3 decision tree by sequentially evaluating feature values against stored thresholds until a terminal leaf node is reached.
predict_probability: Aggregates the raw scores (logits) across all trees in the ensemble, applies the learning rate, and maps the final output to a logistic probability ranging from 0.0 to 1.0 via the Sigmoid function (including numerical protection against overflow/underflow).
calculate_feature_importance: Computes relative feature importance (0.0 to 1.0) by aggregating the structural gain accumulated by each variable across the entire ensemble.

# 🔷 3. Static Quantile Pre-Binning
The find_split_subset_fast function and the initial training phase implement Static Quantile Pre-Binning: prior to boosting, historical feature values are sorted and binned into quantitative buckets. This dramatically accelerates the search for optimal split points during tree construction, significantly reducing computational overhead.

# 🔷 4. The Training Pipeline
This is the core of the library, executing an iterative boosting loop that includes:
1. Row and Column Subsampling: Supports random sampling of instances and features to mitigate overfitting.
2. Gradient Computation: Computes first-order gradients and second-order Hessians based on binary cross-entropy loss.
3. Depth-3 Tree Construction: Progressively identifies optimal splits level by level using XGBoost regularization criteria.
4. Early Stopping & Validation: Automatically carves out a validation subset and halts training if the validation loss fails to improve over a specified number of rounds, subsequently rolling back to the optimal tree count.

# 🔷 Constraints to Consider

🔹 Architectural & Complexity Limitations (Fixed Depth of 3)
The tree is hardcoded with a fixed depth of 3 (XGBTreeDepth3), meaning it can evaluate a maximum of 3 levels of decisions (up to 8 terminal leaves). This can result in an inability to capture complex interactions. In financial markets, complex patterns often require deeper trees to combine multiple simultaneous conditions. A depth of 3 severely limits the learning capacity for advanced non-linear relationships.

🔹 Computational & Execution Limitations
Training a Gradient Boosting model requires a high volume of computations (nested loops for scanning matrices, calculating quantiles, sorting arrays, and evaluating gradients). Increasing the number of trees, feature matrix size, or number of bins too much will cause the script to abort due to exceeding the maximum execution loop limit allowed per single script (typically a few tens of thousands of operations before timing out).

Validation splits data by simply taking a portion of the rows. In financial time series, this can cause Data Leakage if training and validation data mix without strictly respecting the chronological sequence (the model might "peek" into the future if a Walk-Forward or Time-Series Split approach is not used).

Without a rigorous Out-Of-Sample (OOS) test set, a model trained directly on past prices will easily tend to find spurious correlations (market "noise" rather than real signals), failing miserably when applied to future real-time data.

🔹 Technical Rationale for Design Choices
There are very specific technical reasons why advanced features like dynamic Walk-Forward or continuous Rolling Retraining have not been natively integrated into this library:
The Computational Bottleneck: A true Walk-Forward or Rolling Retraining (retraining the model bar-by-bar or across rolling time blocks) requires repeating the entire training process—quantile calculation, matrix scanning, iterative tree construction—hundreds or thousands of times on massive historical datasets. Continuous retraining would immediately trigger an Execution Timeout error.
Memory & Historical Data Architecture: Managing matrices and historical arrays carries strict performance constraints. Accessing past data from hundreds of bars while applying complex temporal slicing logic rapidly consumes the heap memory allocated for the script, slowing down or freezing the chart.

More about this library on my TradingView profile: [https://www.tradingview.com/u/thequantscience/#published-scripts]
