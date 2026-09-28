# 🎚️ DBSCAN Parameter Tuning - Quick Reference Guide

## ⚡ TL;DR (Too Long; Didn't Read)

### Start Here (30 seconds)
```python
# Default settings
eps = 0.5
min_samples = 2 * number_of_features  # For 5 features: min_samples = 10

dbscan = DBSCAN(eps=eps, min_samples=min_samples)
clusters = dbscan.fit_predict(X_scaled)

# If results aren't good, see "Troubleshooting" below
```

---

## 📊 Quick Parameter Reference

| Parameter | What It Does | Typical Range | Default Start |
|-----------|-------------|-----------------|-----------------|
| **eps** | Neighborhood radius | 0.3 - 1.5 | 0.5 |
| **min_samples** | Min points for core | 2×D to 4×D | 2×D |

**D** = number of features in your data

---

## 🔧 Tuning Decision Tree

```
START: Run with eps=0.5, min_samples=2×D

        ↓ Check Results ↓

    ┌───────────────────┬───────────────────┬───────────────────┐
    ↓                   ↓                   ↓
Too Many Noise      Not Enough Clusters  Too Many Clusters
(>20%)              (<2)                  (>10)

    ↓                   ↓                   ↓
  INCREASE ε        DECREASE ε          INCREASE ε & min_samples
  Try: +0.1         Try: -0.1           Try: +0.15 & +2

    ↓                   ↓                   ↓
Re-test          Re-test             Re-test
```

---

## 🎯 Common Scenarios & Quick Fixes

### ❌ Problem: "All points are noise (-1)" or "Everything in one cluster"

**Most Likely Cause:** Epsilon completely wrong

**Quick Fix:**
```python
# Try these in order:
eps_values = [0.2, 0.3, 0.5, 0.7, 1.0]

for eps in eps_values:
    dbscan = DBSCAN(eps=eps, min_samples=10)
    clusters = dbscan.fit_predict(X_scaled)
    n_clusters = len(set(clusters)) - (1 if -1 in clusters else 0)
    n_noise = list(clusters).count(-1)
    print(f"ε={eps}: {n_clusters} clusters, {n_noise} noise")

# Pick the one with reasonable balance
```

### ❌ Problem: "Too many noise points (>30%)"

**Likely Cause:** Epsilon too small OR min_samples too high

**Quick Fix - Option 1 (Recommended):**
```python
# Increase epsilon gradually
new_eps = current_eps + 0.1

dbscan = DBSCAN(eps=new_eps, min_samples=10)
clusters = dbscan.fit_predict(X_scaled)
```

**Quick Fix - Option 2:**
```python
# Decrease min_samples (less strict)
new_min_samples = current_min_samples - 2

dbscan = DBSCAN(eps=0.5, min_samples=new_min_samples)
clusters = dbscan.fit_predict(X_scaled)
```

### ❌ Problem: "Too many clusters (10+), hard to interpret"

**Likely Cause:** Epsilon too small

**Quick Fix:**
```python
# Increase epsilon to merge nearby clusters
new_eps = current_eps + 0.2

dbscan = DBSCAN(eps=new_eps, min_samples=10)
clusters = dbscan.fit_predict(X_scaled)
# Should have fewer, larger clusters
```

### ❌ Problem: "Silhouette score is negative"

**Likely Cause:** Clusters overlap or are poorly separated

**Quick Fixes (in order):**
```python
# Fix 1: Adjust epsilon
eps_values = [0.3, 0.4, 0.5, 0.6, 0.7]

# Fix 2: Increase min_samples (stricter clustering)
dbscan = DBSCAN(eps=0.5, min_samples=15)

# Fix 3: Check if data is standardized
# Run StandardScaler before clustering
```

---

## 🧮 The Math (Simple Explanation)

### Distance Calculation (Euclidean)
```
Distance between Point A and Point B:
d = √[(a₁-b₁)² + (a₂-b₂)² + ... + (aₙ-bₙ)²]

If d ≤ ε: Points are neighbors
If d > ε:  Points are NOT neighbors
```

### Core Point Check
```
If (number of neighbors) ≥ min_samples:
    → Point is CORE POINT
    → Start/expand cluster
Else:
    → Check if close to core point
    → If yes: BORDER point (in cluster)
    → If no: NOISE point (-1)
```

---

## 📈 Using K-Distance Graph

### What It Shows
```
Distance
    │     ╱╱╱╱╱╱╱╱  ← Outliers (few neighbors)
    │   ╱╱╱╱
    │ ╱╱ ← "Knee" (good ε)
    │╱
    └──────────────── Points
```

### How to Use It
```python
from sklearn.neighbors import NearestNeighbors
import numpy as np

k = min_samples  # Use your min_samples value

neighbors = NearestNeighbors(n_neighbors=k)
neighbors.fit(X_scaled)
distances, _ = neighbors.kneighbors(X_scaled)
distances = np.sort(distances[:, k-1])

plt.plot(distances)
plt.axhline(y=0.5, color='r', linestyle='--')  # Try different values
plt.show()

# The "knee" point is a good epsilon candidate
```

---

## ✅ Quick Quality Checks

After clustering, ask:

1. **Is the noise ratio reasonable?**
   ```python
   noise_pct = (clusters == -1).sum() / len(clusters) * 100
   print(f"Noise: {noise_pct:.1f}%")
   # Good: 1-5%, Acceptable: 5-15%, Too high: >30%
   ```

2. **Are there reasonable number of clusters?**
   ```python
   n_clusters = len(set(clusters)) - (1 if -1 in clusters else 0)
   print(f"Clusters: {n_clusters}")
   # For most data: 2-5 clusters is often good
   ```

3. **Is silhouette score decent?**
   ```python
   from sklearn.metrics import silhouette_score
   mask = clusters != -1
   score = silhouette_score(X_scaled[mask], clusters[mask])
   print(f"Silhouette: {score:.3f}")
   # >0.5 is good, >0.7 is excellent
   ```

---

## 🧪 Quick Testing Template

Copy-paste this to test multiple parameters:

```python
from sklearn.metrics import silhouette_score

# Test configurations
test_configs = [
    {'eps': 0.3, 'min_samples': 8},
    {'eps': 0.3, 'min_samples': 10},
    {'eps': 0.5, 'min_samples': 8},
    {'eps': 0.5, 'min_samples': 10},  # Current best guess
    {'eps': 0.5, 'min_samples': 12},
    {'eps': 0.7, 'min_samples': 10},
]

print(f"{'ε':<6} {'min_s':<8} {'Clusters':<10} {'Noise':<8} {'Silhouette':<12} {'Rating'}")
print("-" * 70)

for config in test_configs:
    dbscan = DBSCAN(**config)
    clusters = dbscan.fit_predict(X_scaled)
    
    n_clusters = len(set(clusters)) - (1 if -1 in clusters else 0)
    n_noise = (clusters == -1).sum()
    noise_pct = n_noise / len(clusters) * 100
    
    mask = clusters != -1
    if n_clusters > 1 and mask.sum() > 1:
        silhouette = silhouette_score(X_scaled[mask], clusters[mask])
    else:
        silhouette = -1
    
    # Simple rating
    if 0.5 < silhouette <= 0.7 and 1 < noise_pct < 10:
        rating = "✓ GOOD"
    elif silhouette > 0.7 and noise_pct < 5:
        rating = "✓✓ EXCELLENT"
    else:
        rating = "✗ Poor"
    
    print(f"{config['eps']:<6.1f} {config['min_samples']:<8} {n_clusters:<10} "
          f"{noise_pct:<8.1f} {silhouette:<12.4f} {rating}")

# Choose the best one!
```

---

## 🚀 Pro Tips

### Tip 1: Use K-Distance Graph First
```python
# Always start with k-distance graph analysis
# It gives you a data-driven starting point for epsilon

# Don't just guess!
```

### Tip 2: Think About Your Data
```python
# Does your dataset have:
# - Natural clusters? → Use smaller ε
# - Continuous variation? → Use larger ε
# - Multiple scales? → May need preprocessing

# Think before you tune!
```

### Tip 3: Noise is Information
```python
# Don't dismiss noise points as failures!
# They often represent:
# - Valuable anomalies (fraud, VIPs)
# - Data entry errors
# - Unusual patterns worth investigating

# Analyze noise, don't ignore it!
```

### Tip 4: Cross-Validate
```python
# Run multiple times with slightly different ε values
# If results are similar, you've found stable clusters
# If results are very different, clustering is fragile

# Check for stability!
```

### Tip 5: Visualize!
```python
# Always visualize your clusters with PCA
# Sometimes metrics lie, but visual inspection doesn't

# A picture is worth 1000 metrics!
```

---

## ⚠️ When DBSCAN Doesn't Work Well

DBSCAN might struggle with:

1. **Clusters with Varying Densities**
   - Solution: Use HDBSCAN instead
   ```python
   # pip install hdbscan
   from hdbscan import HDBSCAN
   clusterer = HDBSCAN(min_cluster_size=10)
   ```

2. **High-Dimensional Data (>10 dimensions)**
   - Solution: Use dimensionality reduction first
   ```python
   from sklearn.decomposition import PCA
   pca = PCA(n_components=5)
   X_reduced = pca.fit_transform(X_scaled)
   # Then apply DBSCAN
   ```

3. **No Natural Clusters**
   - Solution: Maybe data doesn't have clusters!
   - Check with silhouette score on multiple algorithms

---

## 📚 Parameter Presets for Common Scenarios

### Scenario 1: Customer Data (like your assignment)
```python
min_samples = 10  # 2 × 5 features
eps = 0.5         # Start here, adjust based on k-distance graph
```

### Scenario 2: Geographic Data (lat/long)
```python
min_samples = 5
eps = 0.01  # Much smaller (coordinates are close together)
```

### Scenario 3: Text Embeddings
```python
min_samples = 10
eps = 0.3  # Embeddings are normalized to unit sphere
```

### Scenario 4: Time Series Features
```python
min_samples = 15  # More strict for time series
eps = 0.8  # Usually larger for normalized time series
```

---

## 🎓 Learning Progression

### Level 1: Beginner (You are here)
- ✅ Understand eps and min_samples
- ✅ Use k-distance graph
- ✅ Tune by trial and error
- ✅ Check silhouette score

### Level 2: Intermediate
- Learn about core vs border vs noise points
- Understand algorithm time complexity O(n²)
- Use different distance metrics (Manhattan, Chebyshev)
- Handle high-dimensional data with PCA

### Level 3: Advanced
- Implement DBSCAN from scratch
- Use HDBSCAN for varying densities
- Implement incremental/streaming clustering
- Use DBSCAN for anomaly detection

---

## 🆘 When to Ask for Help

Ask for help if:
- Silhouette score is consistently negative
- Only 1-2 data points form clusters
- 80%+ of points are noise
- Results don't make business sense

Prepare:
```python
# Share this info
print(f"Data shape: {X.shape}")
print(f"Feature ranges: {X.min()} to {X.max()}")
print(f"ε={eps}, min_samples={min_samples}")
print(f"Clusters: {n_clusters}, Noise: {n_noise}")
print(f"Silhouette: {silhouette_score}")
```

---

## 📞 Quick Reference Cheat Sheet

```
INCREASE ε when:
├─ Too many noise points
├─ Want fewer clusters
└─ Clusters are fragmented

DECREASE ε when:
├─ Everything merged together
├─ Want more clusters
└─ Not enough separation

INCREASE min_samples when:
├─ Too sensitive to noise
├─ Want stricter clustering
└─ Have large dataset

DECREASE min_samples when:
├─ Can't form clusters
├─ Epsilon already good
└─ Have small clusters you want to keep
```

---

## 🎉 You're Ready!

Use this guide to:
1. Start with the defaults
2. Generate k-distance graph
3. Adjust based on your data
4. Evaluate results
5. Iterate until satisfied

**Remember:** There's no one "perfect" answer—choose parameters that make business sense!

---

**Last Updated: 2024**  
**Happy Clustering! 🚀**
