# Contributing to ALDF-EAM

Thank you for your interest in contributing to ALDF-EAM! This document provides guidelines for contributing to the project.

## Table of Contents
1. [Code of Conduct](#code-of-conduct)
2. [How Can I Contribute?](#how-can-i-contribute)
3. [Development Setup](#development-setup)
4. [Pull Request Process](#pull-request-process)
5. [Coding Standards](#coding-standards)
6. [Testing Requirements](#testing-requirements)

---

## Code of Conduct

This project follows the standard open-source code of conduct:
- Be respectful and inclusive
- Welcome newcomers and help them learn
- Focus on constructive criticism
- Assume good intentions

## How Can I Contribute?

### 1. Report Bugs
- Use GitHub Issues to report bugs
- Include: Python version, OS, error messages, minimal reproduction steps
- Label as `bug`

### 2. Suggest Enhancements
- Use GitHub Issues to suggest new features
- Explain the use case and expected behavior
- Label as `enhancement`

### 3. Add New Detection Methods
We welcome contributions of new methods (M7, M8, etc.)! Requirements:
- Method must have clear activation conditions based on S-vector
- Must produce either DT-level residuals or consumer-level risk scores
- Include uncertainty quantification
- Provide test cases

**Example:** Implementing Method M7 (Time-Series Anomaly Detection)

```python
class MethodM7:
    """
    Time-series anomaly detection using LSTM autoencoders.
    
    Activation: B > 0.65 AND D >= 0.70
    Output: Per-consumer anomaly score [0, 1]
    """
    
    def activation_condition(self, s_vector):
        return s_vector['B'] > 0.65 and s_vector['D'] >= 0.70
    
    def execute(self, consumer_data, s_vector):
        # Implementation
        pass
```

### 4. Improve Documentation
- Fix typos, clarify explanations
- Add examples or tutorials
- Translate documentation (especially for India deployment contexts)

### 5. Pilot Real-World Data
If you have access to real AMI data from a utility:
- Run ALDF-EAM on real feeders
- Document S-vector calibration process
- Share anonymized case study results
- Contribute to validation dataset repository

---

## Development Setup

### 1. Fork and Clone
```bash
# Fork the repository on GitHub, then:
git clone https://github.com/YOUR_USERNAME/ALDF-EAM.git
cd ALDF-EAM

# Add upstream remote
git remote add upstream https://github.com/ORIGINAL_OWNER/ALDF-EAM.git
```

### 2. Create Virtual Environment
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
pip install -r requirements-dev.txt  # Development dependencies
```

### 3. Create Feature Branch
```bash
git checkout -b feature/your-feature-name
```

---

## Pull Request Process

### Before Submitting
1. ✅ **Run tests:** `pytest tests/`
2. ✅ **Run linter:** `flake8 src/`
3. ✅ **Format code:** `black src/`
4. ✅ **Type check:** `mypy src/`
5. ✅ **Update documentation** if adding features
6. ✅ **Add tests** for new functionality

### Submitting
1. Push to your fork: `git push origin feature/your-feature-name`
2. Open Pull Request on GitHub
3. Fill in the PR template (see below)
4. Link any related issues

### PR Template
```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature (new detection method, integration, etc.)
- [ ] Documentation update
- [ ] Performance improvement

## Testing
- [ ] Tests pass locally
- [ ] Added tests for new functionality
- [ ] Tested on synthetic dataset
- [ ] (Optional) Tested on real AMI data

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-reviewed the code
- [ ] Commented complex logic
- [ ] Updated documentation
- [ ] No breaking changes (or documented if unavoidable)
```

---

## Coding Standards

### Python Style
- Follow PEP 8
- Use `black` for formatting (line length: 100)
- Use type hints for function signatures
- Write docstrings (Google style)

### Example:
```python
def compute_s_vector(
    observability_subindicators: Dict[str, float],
    data_quality_subindicators: Dict[str, float],
    weights: Dict[str, float]
) -> Dict[str, float]:
    """
    Compute the System State Vector S = [O, D, T, B].
    
    Args:
        observability_subindicators: Dict with keys c_F, c_DT, c_C, c_TOP, c_SYNC
        data_quality_subindicators: Dict with keys d_compl, d_uptime, d_tamper, d_health
        weights: Dict with weights for each sub-indicator
        
    Returns:
        Dict containing O, D, T, B indices in range [0.0, 1.0]
        
    Raises:
        ValueError: If sub-indicators not in valid range [0, 1]
    """
    # Implementation
    pass
```

### Code Organization
```
src/
├── aldf_eam/
│   ├── __init__.py
│   ├── s_vector.py          # S-vector computation
│   ├── data_quality_gate.py # Stage A/Transitional/Full Analytics logic
│   ├── methods/
│   │   ├── __init__.py
│   │   ├── m1_energy_balance.py
│   │   ├── m2_proxy_inference.py
│   │   ├── m3_technical_loss.py
│   │   ├── m4_peer_grouping.py
│   │   ├── m5_dbscan.py
│   │   └── m6_gnn.py
│   ├── bayesian.py          # Bayesian inference and fusion
│   ├── feedback.py          # Feedback loop
│   └── utils.py             # Helper functions
├── tests/
│   ├── test_s_vector.py
│   ├── test_methods.py
│   └── test_integration.py
└── docs/
    └── implementation.md
```

---

## Testing Requirements

### Unit Tests
All new methods must have unit tests covering:
- Activation condition logic
- Normal operation
- Edge cases (missing data, extreme values)
- Error handling

### Example:
```python
def test_method_m1_activation():
    """Test M1 activates only when O >= 0.75 and c_F = 1"""
    s_vector_valid = {'O': 0.80, 'D': 0.70, 'T': 0.50, 'B': 0.40, 'c_F': 1}
    assert MethodM1().activation_condition(s_vector_valid) == True
    
    s_vector_no_feeder = {'O': 0.80, 'D': 0.70, 'T': 0.50, 'B': 0.40, 'c_F': 0}
    assert MethodM1().activation_condition(s_vector_no_feeder) == False
```

### Integration Tests
Test complete pipeline on synthetic dataset:
- S-vector computation
- Gate transitions (Stage A → Transitional → Full Analytics)
- Method execution
- Score fusion
- Performance metrics (IHR, FPR, F1, AUC-ROC)

### Performance Benchmarks
For new methods, document:
- Computational complexity (Big-O)
- Memory usage
- Runtime on synthetic dataset (60 consumers, 60 days)

---

## Areas for Contribution

### High Priority
1. **Real-world validation datasets** (anonymized utility data)
2. **Method M7+** (new detection algorithms)
3. **MDMS integration adapters** (for commercial platforms)
4. **Visualization dashboard** (Streamlit/Dash app)

### Medium Priority
5. **Alternative Bayesian engines** (beyond PyMC)
6. **Distributed computing** (Spark/Dask for 100K+ consumers)
7. **API development** (REST API for field dispatch integration)
8. **Mobile app** (field inspector interface)

### Nice to Have
9. **Translations** (Hindi, Telugu, Tamil for India deployment)
10. **Tutorial videos** (YouTube walkthrough)
11. **Comparison benchmarks** (vs. published NTL methods)

---

## Recognition

Contributors will be:
- Listed in `CONTRIBUTORS.md`
- Acknowledged in paper citations (if contribution is substantial)
- Invited to co-author case studies (for real-world deployments)

---

## Questions?

- **Technical questions:** Open a GitHub Discussion
- **Bugs:** Open a GitHub Issue
- **Feature proposals:** Open a GitHub Issue with `enhancement` label
- **Private inquiries:** Email [YOUR_EMAIL]

---

**Thank you for contributing to ALDF-EAM!** 🎉

Together, we're solving the AMI Maturity Gap and helping utilities reduce losses at every stage of infrastructure deployment.
