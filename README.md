# Self-Supporting Code: A Resilience Design Pattern

## Abstract

Self-Supporting Code is a resilience design pattern where system stability emerges from the structural composition of components rather than external orchestration. Inspired by Leonardo da Vinci's self-supporting bridge—which stands through the geometric interlock of its beams without nails or ropes—this pattern embeds resilience into the architecture itself. Each module enforces local invariants and provides fallback behaviors, creating a system that maintains correctness and degrades gracefully under failure conditions.

## Motivation

Modern distributed systems depend heavily on external resilience mechanisms: orchestrators restart failed services, retry logic masks transient errors, and circuit breakers prevent cascade failures. While effective, these approaches share a fundamental limitation: they are **reactive and external** to the components they protect.

Self-Supporting Code addresses this by making resilience **intrinsic and structural**. Just as da Vinci's bridge achieves stability through the arrangement and interlock of its components, self-supporting systems achieve resilience through:

- **Local invariant enforcement**: Each component guarantees its own correctness
- **Fallback composition**: Degraded behaviors are designed in, not bolted on
- **Mutual verification**: Components validate their interactions at boundaries
- **Autonomous recovery**: Failures trigger local correction loops, not global interventions

This shift from external scaffolding to internal structure creates systems that are resilient by design rather than by supplementation.

## Pattern Definition

### Intent
Design software components that maintain their own stability and correctness through explicit invariants and fallback paths, enabling graceful degradation without centralized coordination.

### Structure

```
┌─────────────────────────────────────┐
│  Self-Supporting Component          │
├─────────────────────────────────────┤
│                                     │
│  ┌─────────────────────────────┐   │
│  │   Input Verification        │   │
│  │   (Invariant Guards)        │   │
│  └──────────┬──────────────────┘   │
│             │                       │
│             ▼                       │
│  ┌─────────────────────────────┐   │
│  │   Core Logic                │   │
│  └──────────┬──────────────────┘   │
│             │                       │
│             ▼                       │
│  ┌─────────────────────────────┐   │
│  │   Output Verification       │   │
│  │   (Invariant Guards)        │   │
│  └──────────┬──────────────────┘   │
│             │                       │
│             ├─── Success ──────────>│
│             │                       │
│             └─── Failure            │
│                   │                 │
│                   ▼                 │
│  ┌─────────────────────────────┐   │
│  │   Fallback Path             │   │
│  │   (Degraded Operation)      │   │
│  └─────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

**Key Components:**

1. **Invariant Guards**: Preconditions and postconditions that define valid states
2. **Core Logic**: Primary operation implementing business requirements
3. **Fallback Path**: Alternative behavior that maintains correctness with degraded functionality
4. **Verification Layer**: Continuous checking of component contracts
5. **Recovery Loop**: Local mechanism to restore valid state after failures

### Properties

A self-supporting component exhibits:

- **Autonomy**: Operates correctly without external coordination
- **Bounded Failure**: Errors are contained and do not propagate
- **Explicit Contracts**: Interface guarantees are enforced, not assumed
- **Graceful Degradation**: Reduced capability rather than complete failure
- **Idempotency**: Safe to retry without side effects
- **Observable State**: Health and degradation are visible to observers

## Implementation Patterns

### 1. Invariant-Guarded Operations

```python
from dataclasses import dataclass
from typing import Optional
import logging

@dataclass
class BankAccount:
    """Account with invariant: balance must never go negative."""
    account_id: str
    balance: float
    
    def __post_init__(self):
        self._enforce_invariant()
    
    def _enforce_invariant(self) -> None:
        """Verify the fundamental account invariant."""
        assert self.balance >= 0, f"Invariant violated: negative balance {self.balance}"
    
    def withdraw(self, amount: float) -> bool:
        """Attempt withdrawal with invariant protection."""
        if amount < 0:
            logging.warning(f"Invalid withdrawal amount: {amount}")
            return False
        
        new_balance = self.balance - amount
        
        if new_balance < 0:
            # Fallback: reject transaction rather than violate invariant
            logging.info(f"Withdrawal {amount} rejected - insufficient funds")
            return False
        
        self.balance = new_balance
        self._enforce_invariant()
        return True
```

### 2. Circuit Breaker with Fallback Cache

```python
import time
from typing import Optional, Callable, TypeVar, Dict

T = TypeVar('T')

class CircuitBreaker:
    """Circuit breaker that provides cached fallbacks during failures."""
    
    def __init__(self, failure_threshold: int = 3, recovery_timeout: float = 60.0):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.open_until = 0.0
        self.cache: Dict[str, T] = {}
    
    def is_open(self) -> bool:
        """Check if circuit is open (blocking calls)."""
        if time.time() >= self.open_until:
            # Recovery timeout expired, allow retry
            self.failure_count = 0
            return False
        return self.failure_count >= self.failure_threshold
    
    def call(self, key: str, fn: Callable[[], T]) -> Optional[T]:
        """Execute function with circuit breaker and cache fallback."""
        
        # If circuit is open, return cached value immediately
        if self.is_open():
            logging.warning(f"Circuit open, using cache for {key}")
            return self.cache.get(key)
        
        try:
            result = fn()
            # Success: reset failure count and update cache
            self.failure_count = 0
            self.cache[key] = result
            return result
            
        except Exception as e:
            # Failure: increment counter and fall back to cache
            self.failure_count += 1
            logging.error(f"Call failed ({self.failure_count}/{self.failure_threshold}): {e}")
            
            if self.failure_count >= self.failure_threshold:
                self.open_until = time.time() + self.recovery_timeout
                logging.warning(f"Circuit opened for {self.recovery_timeout}s")
            
            # Fallback to cached value
            cached = self.cache.get(key)
            if cached is not None:
                logging.info(f"Returning cached value for {key}")
                return cached
            
            # No cache available
            raise
```

### 3. Self-Verifying Data Pipeline Stage

```python
from typing import List, Callable, Any
from dataclasses import dataclass

@dataclass
class PipelineStage:
    """Pipeline stage with input/output verification and error recovery."""
    name: str
    transform: Callable[[List[Any]], List[Any]]
    input_invariant: Callable[[List[Any]], bool]
    output_invariant: Callable[[List[Any]], bool]
    
    def process(self, data: List[Any]) -> List[Any]:
        """Process data with full verification and fallback."""
        
        # Verify input invariant
        if not self.input_invariant(data):
            logging.error(f"{self.name}: Input invariant violated")
            return self._fallback_filter(data)
        
        try:
            result = self.transform(data)
            
            # Verify output invariant
            if not self.output_invariant(result):
                logging.error(f"{self.name}: Output invariant violated")
                return self._fallback_filter(data)
            
            return result
            
        except Exception as e:
            logging.error(f"{self.name}: Transform failed: {e}")
            return self._fallback_filter(data)
    
    def _fallback_filter(self, data: List[Any]) -> List[Any]:
        """Fallback: pass through only items meeting input invariant."""
        return [item for item in data if self._item_valid(item)]
    
    def _item_valid(self, item: Any) -> bool:
        """Check if single item meets basic validity requirements."""
        try:
            return self.input_invariant([item])
        except:
            return False


# Example usage
def normalize_ages(records: List[dict]) -> List[dict]:
    """Normalize age field in records."""
    return [
        {**r, 'age': max(0, min(150, r['age']))} 
        for r in records
    ]

age_normalizer = PipelineStage(
    name="age_normalizer",
    transform=normalize_ages,
    input_invariant=lambda data: all('age' in r for r in data),
    output_invariant=lambda data: all(0 <= r['age'] <= 150 for r in data)
)
```

## Comparison to Existing Resilience Patterns

| Pattern | Focus | Scope | Recovery | Self-Supporting Extension |
|---------|-------|-------|----------|---------------------------|
| **Circuit Breaker** | Prevent calls to failing dependencies | Service boundaries | Stop calling until recovery | + Local fallback paths + Cached degraded responses |
| **Bulkhead** | Isolate resources to contain failures | Resource pools | Limit blast radius | + Mutual verification at boundaries + Coordinated degradation |
| **Retry** | Repeat failed operations | Individual operations | Exponential backoff | + Idempotency guarantees + Fallback after retry exhaustion |
| **Health Check** | Detect unhealthy instances | Service instances | External routing decisions | + Self-diagnosis + Autonomous recovery actions |
| **Chaos Engineering** | Test resilience through fault injection | Entire system | Identify weaknesses | + Self-verification in production + Automated adaptation |

**Key Distinction**: Traditional patterns are **mechanisms**—tools you apply to add resilience. Self-Supporting Code is a **structural principle**—resilience emerges from how components are designed and composed.

## Applicability

Self-Supporting Code is most valuable in:

### Strong Fit
- **Microservices architectures**: Independent services with clear boundaries
- **Data processing pipelines**: Multi-stage transformations with verifiable invariants
- **Edge computing**: Systems operating with intermittent connectivity
- **Financial systems**: Operations requiring strong correctness guarantees
- **Long-running batch processes**: Jobs that must be resumable and verifiable

### Moderate Fit
- **API gateways**: Request routing with fallback behaviors
- **Cache layers**: Systems with primary and fallback data sources
- **Event processing**: Stream processors with exactly-once semantics

### Poor Fit
- **Monolithic applications**: Centralized recovery may be simpler and sufficient
- **Simple CRUD applications**: Overhead outweighs benefits
- **Prototype systems**: Upfront design cost not justified
- **Highly coupled systems**: Cannot establish clear component boundaries

## Design Checklist

When implementing self-supporting components:

### 1. Define Invariants
- [ ] Identify non-negotiable correctness properties
- [ ] Express as executable assertions
- [ ] Document why each invariant matters
- [ ] Distinguish critical vs. performance invariants

### 2. Design Fallback Paths
- [ ] Define degraded-but-correct alternative behaviors
- [ ] Ensure fallbacks are simpler than primary paths
- [ ] Verify fallbacks cannot themselves fail catastrophically
- [ ] Make degradation observable

### 3. Ensure Idempotency
- [ ] Operations can be safely retried
- [ ] Use unique request IDs to detect duplicates
- [ ] Make state changes atomic
- [ ] Consider event sourcing for reconstructability

### 4. Implement Verification
- [ ] Add precondition checks at component entry
- [ ] Add postcondition checks before component exit
- [ ] Log invariant violations with context
- [ ] Instrument frequency of fallback activation

### 5. Enable Observability
- [ ] Expose component health metrics
- [ ] Track fallback usage rates
- [ ] Monitor invariant violation patterns
- [ ] Provide diagnostic endpoints

### 6. Test Resilience
- [ ] Unit tests verify invariants hold
- [ ] Integration tests inject faults
- [ ] Chaos tests validate production behavior
- [ ] Load tests verify graceful degradation

## Advanced: Adaptive Self-Support

The patterns described above provide **static resilience**—fixed invariants and predetermined fallbacks. An emerging frontier is **adaptive self-support**, where components evolve their resilience strategies based on observed behavior.

### Adaptive Capabilities

**1. Dynamic Invariant Learning**
- Analyze historical data to discover implicit invariants
- Tighten or relax constraints based on actual system behavior
- Flag anomalies when runtime behavior violates learned patterns

**2. Predictive Degradation**
- Forecast resource exhaustion or load spikes
- Proactively activate fallback paths before failure
- Gradually degrade rather than cliff-edge failures

**3. Fallback Optimization**
- A/B test different fallback strategies
- Select optimal degradation path based on context
- Learn which fallbacks minimize user impact

**4. Autonomous Recovery**
- Identify root causes of repeated failures
- Apply corrective actions (clear cache, restart subprocess)
- Escalate to human operators only when necessary

### Implementation Approach

```
┌─────────────────────────────────────────┐
│  Adaptive Layer (ML/AI)                 │
│  - Anomaly detection                    │
│  - Predictive models                    │
│  - Reinforcement learning               │
└────────────┬────────────────────────────┘
             │ Observes & Adjusts
             ▼
┌─────────────────────────────────────────┐
│  Self-Supporting Foundation             │
│  - Static invariants                    │
│  - Deterministic fallbacks              │
│  - Idempotent operations                │
└─────────────────────────────────────────┘
```

**Principle**: The foundation must be deterministic and correct. The adaptive layer enhances but never replaces structural resilience.

## Limitations and Trade-offs

### Design Complexity
Self-supporting components require upfront investment in:
- Defining precise invariants
- Implementing verification logic
- Designing fallback behaviors
- Testing failure scenarios

**Mitigation**: Start with critical path components; expand coverage incrementally.

### Debugging Challenges
Local recovery loops can mask systemic issues:
- Component repeatedly falls back, hiding upstream problems
- Distributed tracing becomes more complex
- Root cause analysis requires understanding fallback chains

**Mitigation**: Comprehensive observability; emit structured logs at decision points.

### Performance Overhead
Invariant checking and verification add latency:
- Every operation includes guard clauses
- Fallback paths may be slower than primary path
- Redundant validation at component boundaries

**Mitigation**: Use sampling for expensive checks; disable guards in proven-stable code paths with feature flags.

### Redundancy Costs
Fallback mechanisms duplicate functionality:
- Caches shadow primary data stores
- Multiple validation layers check similar conditions
- Recovery logic repeats across components

**Mitigation**: Share verification libraries; use monitoring to identify redundant checks.

## Related Work

- **Design by Contract** (Bertrand Meyer): Preconditions, postconditions, and invariants as first-class design elements
- **Erlang/OTP Supervisors**: Hierarchical recovery with "let it crash" philosophy
- **Chaos Engineering** (Netflix): Resilience through controlled failure injection
- **Antifragile Systems** (Nassim Taleb): Systems that improve under stress
- **Byzantine Fault Tolerance**: Correctness despite arbitrary component failures

## Conclusion

Self-Supporting Code reframes resilience as a structural property rather than a collection of defensive mechanisms. By embedding invariants, fallbacks, and verification into each component, systems achieve stability through composition—like da Vinci's bridge standing through the interlock of its beams.

This pattern complements rather than replaces existing resilience practices. Circuit breakers, bulkheads, and retries remain valuable tools. Self-Supporting Code provides the architectural foundation upon which these mechanisms achieve their full potential.

As systems grow more distributed and autonomous, resilience cannot rely solely on external orchestration. Self-Supporting Code offers a path toward systems that stand on their own—by design.

---

## References

- Nygard, M. (2018). *Release It!* 2nd Edition. Pragmatic Bookshelf.
- Meyer, B. (1997). *Object-Oriented Software Construction*. Prentice Hall.
- Armstrong, J. (2003). "Making reliable distributed systems in the presence of software errors." PhD thesis, KTH.
- Humble, J., et al. (2016). "Chaos Engineering." IEEE Software, 33(3), 35-41.
- Taleb, N. N. (2012). *Antifragile: Things That Gain from Disorder*. Random House.

## Acknowledgments

Inspired by Leonardo da Vinci's self-supporting bridge design and the principle that elegant solutions emerge from structural composition rather than external scaffolding.

---

## Author

**Jesse Li-Yates**  
 [github.com/jegly](https://github.com/jegly)

*November 2025*
