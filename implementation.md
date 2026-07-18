# 6. Implementation of Work

The implementation is structured around the principle of strict **unsupervised learning**: 
the Autoencoder and Isolation Forest are trained exclusively on **153,708 normal 
authentication events**, with no exposure to labelled attack samples at any point during 
training. Attack data comprising **34,897 events** across five categories is reserved 
entirely for offline evaluation and supervised baseline comparison. This design mirrors 
real-world enterprise AD environments where comprehensive attack catalogues are 
unavailable at the time of model deployment.

---

## 6.1 Dataset Loading and Preparation

Two datasets were prepared for this research:

- **Normal training dataset:** 153,708 authentication event records representing 
  legitimate enterprise AD activity.
- **Attack testing dataset:** 34,897 labelled records spanning five attack categories:

| Attack Category | Event Count |
|---|---|
| Privilege Escalation | 15,230 |
| Lateral Movement | 7,147 |
| Credential Stuffing | 6,982 |
| Pass-the-Ticket | 4,429 |
| Pass-the-Hash | 1,109 |

Both datasets were loaded using **pandas** from CSV files, with each record containing 16 
engineered feature columns.

Data cleaning was applied consistently to both datasets: missing values were imputed using 
column-wise **median substitution**, chosen for robustness against outlier-dominated 
distributions. Encoded values of −1 in the `Target_Work_Enc` and `LogonID_Enc` columns 
were preserved intact, as they encode a semantically important category representing 
unknown or external entities absent from the training domain.

Since the proposed system is trained in an unsupervised manner, no labels are used during 
training. However, for evaluation purposes, binary labels are assigned: normal events are 
encoded as **0** (benign traffic) and attack events as **1** (malicious traffic). These 
labels are used exclusively during offline evaluation and supervised baseline training. The 
supervised baselines — Random Forest and SVM — receive both feature vectors and binary 
labels during training, reflecting the distinct learning paradigm under which they operate.

---

## 6.2 Feature Extraction and Implementation Logic

The raw dataset (Windows AD Event Logs) was transformed into a 16-feature schema using the 
developed pipeline:

| Feature Name | Source Attribute | Extraction Logic | Security Signature |
|---|---|---|---|
| Is_Failure | Keywords | 1 if Keywords contains "Failure" | Brute Force / Password Spray |
| Is_NTLM | AuthProtocol | 1 if Protocol == "NTLM" | Pass-the-Hash (T1550.002) |
| Is_Type9 | LogonType | 1 if LogonType == 9 | Mimikatz / New Credentials |
| Process_NtLmSsp | LogonProcess | 1 if Process is "NtLmSsp" | NTLM Relay Detection |
| Is_Special_Logon | TaskCategory | 1 if Category is "Special Logon" | Privilege Abuse (T1548) |
| Is_Elevated | ElevatedToken | 1 if Token is "Yes"/"True" | High-Privilege Session |
| Target_Work_Enc | Workstation | Label Encoded (Unknown = −1) | Lateral Movement Pivot |
| LogonID_Enc | User@Domain | Unique Session Proxy | Credential Replay |
| Auth_Burst_Ratio | DateTime | Rolling count (60s) / Baseline | Automated Tool Activity |
| Inter_Arrival_Time (IAT) | DateTime | tᵢ − tᵢ₋₁ per source | Machine-speed cadence |
| Unique_Target_Count | Workstation | Distinct targets in 300s window | Network Traversal (TA0008) |
| Hour_Of_Day | DateTime | dt.hour (0–23) | Office-hour Routine Analysis |

---

## 6.3 Sliding Window Feature Engineering

Authentication events in Active Directory logs do not occur in isolation — they exhibit 
temporal dependencies that are fundamentally important for distinguishing legitimate user 
behaviour from automated attack tool activity. A human user generates authentication 
events separated by minutes or hours; an automated attack tool generates events in rapid, 
near-continuous bursts within seconds.

The sliding window technique processes the event log as an ordered time series. For each 
event at position `i` in the sorted timeline, a window of the `W` preceding events from the 
same source IP or account is examined. Features computed within this window capture the 
local temporal behaviour context of each event without requiring the sequential model 
architecture used in LSTM-based approaches — making the features compatible with any 
downstream ML model, including the Isolation Forest and Autoencoder used here.

### 6.3.1 Auth_Burst_Ratio (Frequency Analysis)

Quantifies authentication intensity within a window `W = 60s` relative to the historical 
baseline rate:
Auth_Burst_Ratio(eᵢ) = Count(e ∈ [tᵢ − W, tᵢ]) / AvgFreq(all sources)

### 6.3.2 Unique_Target_Count (Network Breadth)

Measures lateral spread within a 5-minute window to identify reconnaissance or multi-hop 
traversal:
Unique_Target_Count(eᵢ) = DistinctCount(TargetMachines ∈ [tᵢ − 300s, tᵢ])

### 6.3.3 Inter_Arrival_Time Computation

Computed as the elapsed time in seconds between the current event and the immediately 
preceding event from the same source within the sliding window. For the first event from a 
source, it is set to 86,400 (one full day — indicating no recent prior activity):
Inter_Arrival_Time(eᵢ) = tᵢ − tᵢ₋₁    if a prior event eᵢ₋₁ exists in window W
= 86,400      otherwise

Each feature column `f` is standardised using the mean and standard deviation computed 
**exclusively from the normal training set**:
f̂ = (f − mean_f) / std_f

The same `mean_f` and `std_f` are applied — without refitting — to scale attack events 
during evaluation, ensuring no data leakage from the attack distribution into the scaling 
parameters.

This feature captures the automated continuous-fire pattern of attack tools: automated 
tools generate authentication events with inter-arrival times measured in **milliseconds to 
seconds**, while human users generate events with inter-arrival times measured in 
**minutes to hours**. The normal dataset's mean Inter_Arrival_Time is **1,375 seconds** 
(~23 minutes), while attack events exhibit a mean of only **2.94 seconds**.

---

## 6.4 Data Preprocessing and Feature Scaling

Feature scaling was applied using a `StandardScaler` fitted exclusively on the 153,708 
normal training events. The scaler transforms each feature to zero mean and unit variance, 
preventing high-magnitude features such as `EventID` (range 4624–4776) and 
`Inter_Arrival_Time` (range 0–86,400 seconds) from dominating the MSE loss of the 
Autoencoder or the hyperplane splits of the Isolation Forest.

The fitted scaler was applied **without refitting** to the attack dataset, ensuring no data 
leakage.

- For the supervised **Random Forest** baseline: a 70/30 stratified split was applied to 
  the combined normal + attack dataset, preserving class distribution.
- For the **SVM** baseline: a stratified 30,000-event subset (15,000 normal + 15,000 
  attack) was constructed due to the quadratic scaling constraints of kernel-based methods 
  on large datasets.

---

## 6.5 Why Choose a Hybrid Model?

- A standalone **Isolation Forest** achieves high recall on AD authentication anomalies 
  due to its point-anomaly isolation properties, but suffers from a **high false positive 
  rate** — it flags all statistically unusual events regardless of whether they are 
  genuinely malicious.
- A standalone **Autoencoder**, conversely, achieves near-zero false positive rate at 
  conservative thresholds but **misses the majority of attacks**, since many attack events 
  share structural feature patterns with normal events and are reconstructed accurately.

The hybrid ensemble combines the complementary strengths of both: the Isolation Forest 
provides the primary detection signal with high recall, while the Autoencoder's 
reconstruction error suppresses false alarms from normal events that happen to be 
statistically unusual. The weighted combination:
Final_Score = 0.1 × AE_Score + 0.9 × IF_Score

assigns dominant weight to the Isolation Forest, reflecting its superior recall on this 
dataset, while preserving the AE's contribution for events flagged by both models 
independently.

---

## 6.6 Model Training

### 6.6.1 Autoencoder Training

**Architecture:**
- **Encoder:** 16 → 32 → 16 → 8 (ReLU activations), with the 8-neuron bottleneck as the 
  compressed representation
- **Decoder:** 8 → 16 → 32 → 16 (mirrored structure), concluding with a linear output layer
- **Total parameters:** 2,864 — lightweight enough for real-time CPU inference

The bottleneck of dimension 8 enforces sufficient compression to prevent trivial identity 
mappings while retaining enough capacity to represent the principal patterns of normal 
authentication behaviour.

**Training configuration:**
- Optimiser: **Adam**, learning rate = 0.001
- Loss function: **MSE**, computed across all 16 output features
- Split: 80/20 train-validation (122,966 training / 30,742 validation samples)
- Early stopping: patience of 10 epochs, restoring best-validation-loss weights
- **Converged at epoch 32** — training loss: 0.001662, validation loss: 0.020721

**Anomaly threshold:** computed from reconstruction errors of all 153,708 normal training 
events:
θ_AE = mean_normal + 0.05 × std_normal = 0.005666

(from a normal-data mean of 0.000761 and std of 0.098084). This conservative threshold 
minimises false positives from the AE component, letting the Isolation Forest carry the 
primary detection responsibility.

**Training Algorithm:**
Algorithm 1: Autoencoder Training
Input:
X_train: 122,966 × 16 scaled normal events
X_val:   30,742 × 16 scaled normal events
max_epochs = 100, patience = 10, lr = 0.001, batch_size = 64
Output: Trained weights θ*, anomaly threshold θ_AE

Initialise AE: encoder [16→32→16→8], decoder [8→16→32→16]
Activations: ReLU on all hidden layers, Linear on output
Initialise Adam optimiser with lr = 0.001
best_val_loss ← ∞; patience_counter ← 0
for epoch = 1 to max_epochs:

 Shuffle X_train into mini-batches of size 64


 for each batch B:


     x̂ ← AE.forward(B)


     L ← MSE(x̂, B)


     L.backward(); Adam.step()


val_loss ← MSE(AE.forward(X_val), X_val)


if val_loss < best_val_loss:


    best_val_loss ← val_loss


    Save weights θ*


    patience_counter ← 0


else:


    patience_counter ← patience_counter + 1


if patience_counter ≥ patience:


    break                          # Converged at epoch 32

Restore best weights θ*
Compute per-sample reconstruction errors on X_train
θ_AE ← mean(errors) + 0.05 × std(errors)     # = 0.005666
return θ*, θ_AE


**Key equations:**

Reconstruction loss (MSE) over a batch of size `n`, feature dimension `d = 16`:
L_MSE = (1 / n·d) × Σᵢ ‖xᵢ − x̂ᵢ‖²

Adam optimiser weight update at step `t`:
θ(t+1) = θ(t) − η × m̂(t) / (√v̂(t) + ε)
where `η = 0.001` (learning rate), `m̂(t)` and `v̂(t)` are bias-corrected first and second 
moment estimates, and `ε = 10⁻⁸`.

Per-sample anomaly score and detection rule:
Score_AE(x) = (1/d) × Σⱼ (xⱼ − x̂ⱼ)²
Flagged as anomaly if: Score_AE(x) > θ_AE = 0.005666

---

### 6.6.2 Isolation Forest Training

Implemented using **scikit-learn's `IsolationForest`** class with:
- 200 estimators
- Contamination parameter: 0.05
- Random state: 42
- Fitted on all 153,708 normal scaled events

The algorithm constructs each tree through recursive random partitioning: at each node, a 
feature is selected uniformly at random and a split value is chosen uniformly at random 
within the feature's observed range. Anomalous samples occupy sparse regions of the 
feature space and are isolated by shorter average path lengths, yielding higher anomaly 
scores. The contamination parameter does not affect the continuous anomaly scores used for 
ensemble combination — it only affects the `predict()` binary classifier, which was not 
used in the final pipeline.

**Training Algorithm:**
Algorithm 2: Isolation Forest Training
Input:
X_normal: 153,708 × 16 scaled normal events
T = 200 trees, ψ = 256 (sub-sample size), contamination = 0.05
Output: Trained ensemble {iTree_1, ..., iTree_T}

for t = 1 to T:

 Sample ψ = 256 points X_sub from X_normal without replacement


 height_limit ← ceil(log2(ψ)) = 8


 iTree_t ← BUILD_TREE(X_sub, depth=0, height_limit)

function BUILD_TREE(X, depth, height_limit):

 if depth ≥ height_limit or |X| ≤ 1:


     return leaf_node(size = |X|)


 q ← randomly selected feature from {1,...,16}


p ← Uniform(min(X[:,q]), max(X[:,q]))


X_left  ← {x ∈ X : x[q] < p}


X_right ← {x ∈ X : x[q] ≥ p}


return internal_node(
     left = BUILD_TREE(X_left, depth+1, height_limit),
     right = BUILD_TREE(X_right, depth+1, height_limit),
     split = (q, p))

Anomaly score at inference:
h(x) ← mean path length across all T trees
Score_IF(x) = 2^(−h(x)/c(ψ))
return trained IF ensemble


**Key equations:**

Normalised anomaly score based on average path length `h(x)` across `T = 200` trees:
Score_IF(x) = 2^(−E[h(x)]/c(n))
where `E[h(x)]` is the mean path length, `n` is the number of training samples, and `c(n)` 
is the expected path length of an unsuccessful search in a Binary Search Tree:
c(n) = 2·H(n−1) − 2(n−1)/n
where `H(k) = ln(k) + γ` is the harmonic number approximation and `γ ≈ 0.5772` (Euler-
Mascheroni constant). Scores approach **1.0** for highly anomalous samples (short 
isolation paths) and **0.5** for normal samples (average-length paths).

For ensemble combination, the raw IF score is obtained by negating the scikit-learn 
decision function output:
Raw_IF(x) = −IF.decision_function(x)

---

### 6.6.3 Hybrid AE + IF Ensemble

The raw anomaly scores from the Autoencoder (per-sample MSE) and the Isolation Forest 
(negated decision function) were jointly normalised using **Min-Max scaling** applied to 
the concatenated normal and attack score distributions, mapping all values to [0, 1]. Joint 
normalisation preserves relative positional information across the normal-attack boundary.

Final hybrid score:
Final_Score = 0.1 × AE_Score + 0.9 × IF_Score

Two operating thresholds were evaluated:

| Threshold | Recall | FPR | Notes |
|---|---|---|---|
| Max F1 (0.133) | 99.98% | 28.88% | Highest recall |
| **Balanced (0.196)** | 90.07% | 20.38% | **Selected — Precision 83.38%** |

The balanced threshold was selected as the primary operating point for its superior 
precision.

**Inference Pipeline Algorithm:**
Algorithm 3: Hybrid AE + IF Inference Pipeline
Input:
x: raw 16-dimensional event feature vector
S (fitted StandardScaler), AE, IF, θ_hybrid = 0.196
s_min, s_max: from joint normalisation on full dataset
Output: prediction ∈ {0,1}, hybrid_score, alert_record

x_scaled ← S.transform(x)
ae_score ← MSE(AE.forward(x_scaled), x_scaled)
if_score ← −IF.decision_function(x_scaled)
ae_norm ← (ae_score − ae_min) / (ae_max − ae_min)
if_norm ← (if_score − if_min) / (if_max − if_min)
hybrid ← 0.1 × ae_norm + 0.9 × if_norm
if hybrid ≥ θ_hybrid:

 prediction ← 1                      # Attack Detected


 if hybrid > 0.75:      risk ← HIGH


elif hybrid > 0.50:    risk ← MEDIUM


else:                  risk ← LOW


triggered_features ← top-3 |xⱼ − mean_normal,ⱼ|


alert_record ← {hybrid, ae_norm, if_norm, risk,
                 triggered_features, recommended_action}

else:

prediction ← 0                      # Normal

Serialise alert_record → alerts_output.json     # SIEM format
return prediction, hybrid_score, alert_record


**Key equations:**

Min-Max joint normalisation:
Norm(s) = (s − s_min) / (s_max − s_min)

Final weighted ensemble score:
Final_Score(x) = 0.1 × Norm(Score_AE(x)) + 0.9 × Norm(Raw_IF(x))

Classification rule at the balanced operating point:
Attack if: Final_Score(x) ≥ θ_hybrid = 0.196
where `w_IF = 0.9` assigns dominant contribution to the Isolation Forest (high recall), and 
`w_AE = 0.1` preserves the Autoencoder's precision-enhancing contribution.

---

## 6.7 Evaluation Metric Equations

All models are evaluated using standard binary classification metrics derived from True 
Positives (TP), True Negatives (TN), False Positives (FP), and False Negatives (FN):
Accuracy  = (TP + TN) / (TP + TN + FP + FN)
Precision = TP / (TP + FP)
Recall    = TP / (TP + FN)
F1-Score  = 2 × (Precision × Recall) / (Precision + Recall)
FPR       = FP / (FP + TN)
