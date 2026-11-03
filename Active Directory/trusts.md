# trusts

[parent-child](trusts/parent-child.md)

[cross-forest](trusts/cross-forest.md)

**I. Domain Trusts Overview**

**Definition:** Trusts establish authentication between domains or forests, allowing users to access resources in other domains.

**Types of Trusts:**

- **Parent-child:** Within the same forest, transitive, bidirectional.
- **Cross-link:** Between child domains, transitive.
- **External:** Between separate forests, non-transitive, uses SID filtering.
- **Tree-root:** Between forest root and new tree root, transitive, bidirectional.
- **Forest:** Between forest root domains, transitive.
- **ESAE:** Used for bastion forests.

**Trust Attributes:**

- **Transitive vs. Non-transitive:**
    - Transitive trusts extend trust to trusted objects of the child domain.
    - Non-transitive trusts only trust the child domain itself.
- **One-way vs. Bidirectional:**
    - One-way: Trusted domain users can access resources in the trusting domain.
    - Bidirectional: Users from both domains can access resources in each other.