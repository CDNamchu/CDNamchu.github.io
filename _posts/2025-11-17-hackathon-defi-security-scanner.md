---
title: "Hackathon Success: Building a DeFi Security Scanner in 48 Hours"
date: 2025-11-17
categories: [Blog]
tags: [Hackathon, Blockchain, Web3, DeFi, Security]
---

# Hackathon Success: Building a DeFi Security Scanner in 48 Hours

Last month, our team participated in a Web3 security hackathon, and what a whirlwind it was! In just 48 hours, we built a DeFi smart contract security scanner that won second place. This post chronicles our journey, technical decisions, and lessons learned.

## The Challenge

**Hackathon:** Web3 Security Innovation Challenge  
**Duration:** 48 hours  
**Team Size:** 4 developers  
**Challenge:** Build a tool that improves smart contract security

Our idea: Create an automated security scanner that analyzes DeFi protocols for common vulnerabilities before deployment.

## Day 1: Planning and Architecture

### Hour 0-4: Brainstorming and Planning

We started by identifying the most critical vulnerabilities in DeFi:
- Reentrancy attacks
- Flash loan exploits
- Access control issues
- Integer overflow/underflow
- Unchecked external calls

**Key Decision:** Focus on automated detection rather than trying to catch everything manually.

### Hour 4-8: Architecture Design

```
User Interface (Streamlit)
        ↓
   Smart Contract Upload
        ↓
   Static Analysis Engine (Slither)
        ↓
   ML-based Pattern Detection
        ↓
   Vulnerability Report Generator
```

**Tech Stack:**
- **Frontend:** Streamlit (rapid prototyping)
- **Analysis:** Slither, Mythril
- **ML:** Pre-trained model for pattern detection
- **Backend:** Python FastAPI
- **Database:** SQLite for caching results

### Hour 8-12: Initial Implementation

First, we built the core analysis pipeline:

```python
import streamlit as st
from slither import Slither
import json

class DeFiSecurityScanner:
    def __init__(self):
        self.vulnerabilities = []
        
    def analyze_contract(self, contract_path):
        """Main analysis pipeline"""
        # Run Slither analysis
        slither = Slither(contract_path)
        
        # Extract contract info
        for contract in slither.contracts:
            self.analyze_functions(contract)
            self.check_defi_patterns(contract)
            self.analyze_dependencies(contract)
        
        return self.generate_report()
    
    def analyze_functions(self, contract):
        """Analyze individual functions for vulnerabilities"""
        for function in contract.functions:
            # Check for reentrancy
            if self.has_reentrancy_risk(function):
                self.vulnerabilities.append({
                    'type': 'Reentrancy',
                    'severity': 'HIGH',
                    'function': function.name,
                    'description': f'Function {function.name} makes external calls before state changes'
                })
            
            # Check for unchecked calls
            if self.has_unchecked_calls(function):
                self.vulnerabilities.append({
                    'type': 'Unchecked Call',
                    'severity': 'MEDIUM',
                    'function': function.name,
                    'description': f'Return value not checked in {function.name}'
                })
```

## Day 1 Evening: First Working Prototype

By hour 12, we had:
- ✅ Basic contract parsing
- ✅ Reentrancy detection
- ✅ Simple web interface
- ❌ ML integration (postponed)
- ❌ DeFi-specific checks (in progress)

### Demo Time Issues

When we tried to demo to each other:
- Streamlit kept crashing with large contracts
- Slither analysis was too slow
- No way to visualize findings

**Emergency pivot:** Cache analysis results and add progress indicators.

```python
import hashlib
import pickle
from pathlib import Path

class ResultCache:
    def __init__(self, cache_dir='.cache'):
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(exist_ok=True)
    
    def get_cache_key(self, contract_code):
        """Generate cache key from contract code"""
        return hashlib.sha256(contract_code.encode()).hexdigest()
    
    def get(self, contract_code):
        """Retrieve cached results"""
        key = self.get_cache_key(contract_code)
        cache_file = self.cache_dir / f"{key}.pkl"
        
        if cache_file.exists():
            with open(cache_file, 'rb') as f:
                return pickle.load(f)
        return None
    
    def set(self, contract_code, results):
        """Cache results"""
        key = self.get_cache_key(contract_code)
        cache_file = self.cache_dir / f"{key}.pkl"
        
        with open(cache_file, 'wb') as f:
            pickle.dump(results, f)
```

## Day 2: Sprint to the Finish

### Hour 13-20: DeFi-Specific Analysis

This is where the magic happened. We implemented DeFi-specific vulnerability checks:

```python
class DeFiPatternDetector:
    def __init__(self):
        self.patterns = []
    
    def check_flash_loan_protection(self, contract):
        """Check for flash loan attack protection"""
        findings = []
        
        for function in contract.functions:
            # Look for missing balance checks after external calls
            if self.calls_external_contract(function):
                if not self.has_balance_validation(function):
                    findings.append({
                        'type': 'Flash Loan Vulnerability',
                        'severity': 'CRITICAL',
                        'function': function.name,
                        'description': 'Function may be vulnerable to flash loan attacks',
                        'recommendation': 'Add balance checks after external calls'
                    })
        
        return findings
    
    def check_price_oracle_manipulation(self, contract):
        """Detect potential oracle manipulation"""
        findings = []
        
        for function in contract.functions:
            # Check if price is fetched from single source
            oracle_calls = self.get_price_oracle_calls(function)
            
            if len(oracle_calls) == 1:
                findings.append({
                    'type': 'Oracle Manipulation Risk',
                    'severity': 'HIGH',
                    'function': function.name,
                    'description': 'Relies on single price oracle',
                    'recommendation': 'Use multiple oracle sources or TWAP'
                })
        
        return findings
    
    def check_slippage_protection(self, contract):
        """Check for slippage protection in swaps"""
        findings = []
        
        for function in contract.functions:
            if 'swap' in function.name.lower():
                # Check for minimum output amount parameter
                has_slippage_param = any(
                    'min' in param.name.lower() 
                    for param in function.parameters
                )
                
                if not has_slippage_param:
                    findings.append({
                        'type': 'Missing Slippage Protection',
                        'severity': 'MEDIUM',
                        'function': function.name,
                        'description': 'Swap function lacks slippage protection',
                        'recommendation': 'Add minAmountOut parameter'
                    })
        
        return findings
```

### Hour 20-24: UI/UX Polish

We realized judges care about presentation. Quick Streamlit improvements:

```python
import streamlit as st
import plotly.graph_objects as go

def create_dashboard(vulnerabilities):
    st.title("🔒 DeFi Security Scanner")
    st.markdown("### Upload your smart contract for security analysis")
    
    # File upload
    uploaded_file = st.file_uploader("Choose a Solidity file", type=['sol'])
    
    if uploaded_file:
        with st.spinner('Analyzing contract...'):
            results = analyze_contract(uploaded_file)
        
        # Summary metrics
        col1, col2, col3, col4 = st.columns(4)
        
        with col1:
            st.metric("Total Issues", len(results['vulnerabilities']))
        with col2:
            critical = sum(1 for v in results['vulnerabilities'] if v['severity'] == 'CRITICAL')
            st.metric("Critical", critical, delta=None, delta_color="inverse")
        with col3:
            high = sum(1 for v in results['vulnerabilities'] if v['severity'] == 'HIGH')
            st.metric("High", high)
        with col4:
            score = calculate_security_score(results)
            st.metric("Security Score", f"{score}/100")
        
        # Vulnerability breakdown chart
        severity_counts = count_by_severity(results['vulnerabilities'])
        fig = go.Figure(data=[go.Pie(
            labels=list(severity_counts.keys()),
            values=list(severity_counts.values()),
            marker_colors=['#ff4444', '#ff8800', '#ffcc00', '#44ff44']
        )])
        st.plotly_chart(fig)
        
        # Detailed findings
        st.markdown("### Detailed Findings")
        for vuln in results['vulnerabilities']:
            severity_color = {
                'CRITICAL': '🔴',
                'HIGH': '🟠',
                'MEDIUM': '🟡',
                'LOW': '🟢'
            }
            
            with st.expander(f"{severity_color[vuln['severity']]} {vuln['type']} in {vuln['function']}"):
                st.write(f"**Severity:** {vuln['severity']}")
                st.write(f"**Description:** {vuln['description']}")
                st.write(f"**Recommendation:** {vuln['recommendation']}")
                
                # Code snippet if available
                if 'code_snippet' in vuln:
                    st.code(vuln['code_snippet'], language='solidity')
```

### Hour 24-36: The Grind

This was the toughest part. We had to:
- Fix bugs found during internal testing
- Handle edge cases (empty contracts, syntax errors)
- Add error handling everywhere
- Write documentation

**Biggest bug:** Memory leak when analyzing large contracts. Solution:

```python
def analyze_with_timeout(contract_path, timeout=30):
    """Analyze contract with timeout protection"""
    import signal
    
    def timeout_handler(signum, frame):
        raise TimeoutError("Analysis exceeded time limit")
    
    signal.signal(signal.SIGALRM, timeout_handler)
    signal.alarm(timeout)
    
    try:
        result = analyze_contract(contract_path)
        signal.alarm(0)  # Cancel alarm
        return result
    except TimeoutError:
        return {
            'error': 'Analysis timeout',
            'message': 'Contract too complex to analyze in time limit'
        }
```

### Hour 36-42: Real-World Testing

We tested on actual DeFi protocols:
- Uniswap V2 - Detected known issues
- SushiSwap - Found 3 medium-severity issues
- Custom vulnerable contract - Caught all intentional vulnerabilities

**Success rate:** 87% true positive, 13% false positive

### Hour 42-48: Final Polish and Presentation

Last 6 hours were all about the pitch:
- Created demo video
- Wrote README
- Prepared slide deck
- Practiced presentation
- Fixed one last critical bug (found at hour 46!)

## The Pitch

Our 5-minute pitch focused on:
1. **Problem:** DeFi hacks cost $3B in 2024
2. **Solution:** Automated scanner with DeFi-specific checks
3. **Demo:** Live analysis of vulnerable contract
4. **Impact:** Could prevent 70% of common DeFi vulnerabilities
5. **Next Steps:** Integration with deployment pipelines

## Results

**🥈 2nd Place!**

**Judge Feedback:**
- ✅ "Addresses real problem in DeFi security"
- ✅ "Impressive technical depth for 48 hours"
- ✅ "Clean, usable interface"
- ⚠️ "Needs more comprehensive testing"
- ⚠️ "Some false positives in analysis"

## Lessons Learned

### Technical Lessons

1. **Start simple, iterate fast** - We almost over-engineered the ML component
2. **Cache everything** - Saved us when demos were slow
3. **Error handling is not optional** - Learned this the hard way
4. **UI matters** - Good presentation influenced judging

### Team Lessons

1. **Clear role division** - Each person had ownership
2. **Regular sync-ups** - Avoided duplicate work
3. **Sleep matters** - The hour 30 bug was from exhaustion
4. **Have fun** - Best moments were debugging together at 3 AM

### Hackathon-Specific Tips

1. **Read the rules carefully** - We almost missed a submission requirement
2. **Demo-driven development** - Build for the 5-minute pitch
3. **Document as you go** - README written at hour 45 was rushed
4. **Test on real examples** - Made our demo compelling

## The Code

Full project: [github.com/team/defi-security-scanner](https://github.com)

Key components:
- `analyzer.py` - Core security analysis
- `defi_patterns.py` - DeFi-specific checks
- `app.py` - Streamlit interface
- `tests/` - Test contracts

## What's Next

Post-hackathon, we're:
- Expanding vulnerability database
- Adding more DeFi protocols
- Improving ML-based detection
- Building CI/CD integration
- Seeking partnerships with auditing firms

## Try It Yourself

Want to run the scanner?

```bash
# Clone the repo
git clone https://github.com/team/defi-security-scanner
cd defi-security-scanner

# Install dependencies
pip install -r requirements.txt

# Run the app
streamlit run app.py
```

## Conclusion

Hackathons are intense, exhausting, and incredibly rewarding. In 48 hours, we went from an idea to a working product that could genuinely help secure DeFi protocols.

The key is balancing ambition with pragmatism - we wanted to build something impressive but had to scope ruthlessly to finish in time. The result was a focused tool that does one thing well.

If you're considering your first hackathon: **Do it.** The learning experience is invaluable, and you'll surprise yourself with what you can build under pressure.

## Team

- **Alice** - Lead Developer, Analysis Engine
- **Bob** - DeFi Patterns, Smart Contract Expert
- **Charlie** - Frontend, UI/UX
- **Me** - Architecture, ML Integration, Coordination

## Resources

- [Slither Documentation](https://github.com/crytic/slither)
- [DeFi Security Best Practices](https://blog.openzeppelin.com/defi-security/)
- [Streamlit Documentation](https://docs.streamlit.io/)
- [Smart Contract Security Verification Standard](https://github.com/securing/SCSVS)

---

*Participating in a hackathon soon? Let's share strategies and lessons learned!*
