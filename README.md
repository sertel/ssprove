# SSProve: A Foundational Framework for Modular Cryptographic Proofs in Coq

This complementary material corresponds to the Coq formalisation of the results
mentioned in the paper.
This README serves as guide to installing this project and finding
correspondence between the claims in the paper and their formal proofs in Coq,
as well as listing the assumptions that the formalisation makes.

## Prerequisites

- Ocaml
- Coq `8.12.0`
- Equations `1.2.3+8.12`
- Mathcomp analysis `0.3.2`
- Coq Extructures `0.2.2`

To build the dependency graph, you can optionally install `graphviz`. On macOS,
`gsed` is additionally required for this.

You can get them from the `opam` package manager for `ocaml`:
```sh
opam repo add coq-released https://coq.inria.fr/opam/released
opam update
opam install coq.8.12.0 coq-equations.1.2.3+8.12 coq-mathcomp-analysis.0.3.2 coq-extructures.0.2.2
```

## Step-by-step compilation guide

Run `make` from this directory to compile all the coq files
(this step is needed for the walkthrough). It should succeed
displaying only warnings.

Run `make graph` to build a graph of dependencies between sources.

## Organisation of the directories

| Directory             | Description                                          |
|-----------------------|------------------------------------------------------|
| `theories/`           | Root of all the Coq files                            |
| `theories/Mon`        | External development coming from "Dijkstra Monads For All" |
| `theories/Relational` | External development coming from "The Next 700 Relational Program |Logics"
| `theories/Crypt`      | This paper                                           |

## Mapping between paper and formalisation

### Package definition and laws

The formalisation of packages can be found in the directory
`theories/Crypt/package`.

The definition of packages can be found in `pkg_core_definition.v`. Note that
the definition is slightly different from the paper, but the differences are
only in naming. `raw_code` is referred to as `raw_program` and similarly
`code` is called `program`. The final version of `SSProve` will account for
this renaming.
Herein, `package I E` is the type of packages with import interface `I` and export
interface `E`.

Package laws, as introduced in the paper, are all stated and proven in
`pkg_composition.v` directly on raw packages.


#### Sequential composition

In Coq, we call `link p1 p2` the sequential composition of `p1` and `p2`:
`p1 ∘ p2`.

```coq
Definition link (p1 p2 : raw_package) : raw_package.
```

Linking is valid if the export and import match:
```coq
Lemma valid_link :
  ∀ L1 L2 I M E p1 p2,
    valid_package L1 M E p1 →
    valid_package L2 I M p2 →
    valid_package (L1 :|: L2) I E (link p1 p2).
```

Associativity is stated as follows:

```coq
Lemma link_assoc :
  ∀ p1 p2 p3,
    link p1 (link p2 p3) =
    link (link p1 p2) p3.
```
It holds directly on raw packages, even if they are ill-formed.

#### Parallel composition

In Coq, we write `par p1 p2` for the parallel composition of `p1` and `p2`:
`p1 || p2`.

```coq
Definition par (p1 p2 : raw_package) : raw_package.
```

The validity of parallel composition can be proven with the following lemma:
```coq
Lemma valid_par :
  ∀ L1 L2 I1 I2 E1 E2 p1 p2,
    Parable p1 p2 →
    valid_package L1 I1 E1 p1 →
    valid_package L2 I2 E2 p2 →
    valid_package (L1 :|: L2) (I1 :|: I2) (E1 :|: E2) (par p1 p2).
```

The `Parable` condition checks that the export interfaces are indeed disjoint.

We have commutativity as follows:
```coq
Lemma par_commut :
  ∀ p1 p2,
    Parable p1 p2 →
    par p1 p2 = par p2 p1.
```
This lemma does not work on arbitrary raw packages, it is requires that the
packages implement disjoint signatures.

Associativity on the other hand is free from this problem:
```coq
Lemma par_assoc :
  ∀ p1 p2 p3,
    par p1 (par p2 p3) = par (par p1 p2) p3.
```

#### Identity package

The identity package is called `ID` in Coq and has the following type:
```coq
Definition ID (I : Interface) : raw_package.
```

Its validity is stated as
```coq
Lemma valid_ID :
  ∀ L I,
    flat I →
    valid_package L I I (ID I).
```

Note the extra `flat I` condition on the interface which essentially forbids
overloading: there cannot be two procedures in `I` that share the same name.
While our interfaces could in theory allow such procedures in general, the
way we build packages forbid us from ever implementing them, hence the
restriction.

The two identity laws are as follows:
```coq
Lemma link_id :
  ∀ L I E p,
    valid_package L I E p →
    flat I →
    trimmed E p →
    link p (ID I) = p.
```

```coq
Lemma id_link :
  ∀ L I E p,
    valid_package L I E p →
    trimmed E p →
    link (ID E) p = p.
```

In both cases, we ask that the package we link the identity package with is
`trimmed`, meaning that it implements *exactly* its export interface and nothing
more. Packages created through our interfaces always verify this property.

#### Interchange between sequential and parallel composition

Finally we prove a law involving sequential and parallel composition
stating how we can interchange them:
```coq
∀ A B C D E F L1 L2 L3 L4 p1 p2 p3 p4,
  ValidPackage L1 B A p1 →
  ValidPackage L2 E D p2 →
  ValidPackage L3 C B p3 →
  ValidPackage L4 F E p4 →
  trimmed A p1 →
  trimmed D p2 →
  Parable p3 p4 →
  par (link p1 p3) (link p2 p4) = link (par p1 p2) (par p3 p4).
```
which can be read as
`(p1 ∘ p3) || (p2 ∘ p4) = (p1 || p2) ∘ (p3 || p4)`.

It once again requires some validity and trimming properties.

### Examples

The PRF example is developed in `theories/Crypt/examples/PRF.v`.
The security theorem is the following:

```coq
Theorem security_based_on_prf :
  ∀ LA A,
    ValidPackage LA
      [interface val #[i1] : chWords → chWords × chWords ] A_export A →
    fdisjoint LA (IND_CPA false).(locs) →
    fdisjoint LA (IND_CPA true).(locs) →
    Advantage IND_CPA A <=
    prf_epsilon (A ∘ MOD_CPA_ff_pkg) +
    statistical_gap A +
    prf_epsilon (A ∘ MOD_CPA_tt_pkg).
```

As we claim in the paper, it bounds the advantage of any adversary to the
game pair `IND_CPA` by the sum of the statistical gap and the advantages against
`MOD_CPA`.

Note that we require here some disjointness of state hypotheses as these are
not enforced by our package definitions and laws.


The ElGamal example is developed in `theories/Crypt/examples/Elgamal.v`
The security theorem is the following:

```coq
Theorem ElGamal_OT (dh_secure : DH_security) : OT_rnd_cipher.
```

`OT_rnd_cipher` is defined in `theories/Crypt/examples/AsymScheme.v`:

 ```coq
Definition OT_rnd_cipher : Prop :=
  ∀ LA A,
    ValidPackage LA [interface val #[challenge_id'] : chPlain → 'option chCipher] A_export A →
    fdisjoint LA (ots_real_vs_rnd true).(locs) →
    fdisjoint LA (ots_real_vs_rnd false).(locs) →
    Advantage ots_real_vs_rnd A = 0.
```

Note that the theorem relies on `dh_secure` which corresponds to the DDH
assumption:

```coq
Definition DH_security : Prop :=
  ∀ LA A,
    ValidPackage LA [interface val #[10] : 'unit → chPubKey × chCipher ] A_export A →
    fdisjoint LA DH_loc →
    AdvantageE DH_rnd DH_real A = 0.
```


### Probabilistic relational program logic

Figure 13 presents a selection of rules for our probabilistic relational
program logic. Most of them can be found in
`theories/Crypt/package/pkg_rhl.v` which provides an interface for use of those
rules directly with `code`.

| Rule in paper | Rule in Coq           |
|---------------|-----------------------|
| reflexivity   | `rreflexivity_rule`   |
| seq           | `rbind_rule`          |
| swap          | `rswap_rule`          |
| eqDistrL      | `rrewrite_eqDistrL`   |
| symmetry      | `rsymmetry`           |

Some rules are for now only proven in the logic but have not been lifted
to packages, they can be found in `theories/Crypt/rules/UniformStateProb.v`:

| Rule in paper | Rule in Coq             |
|---------------|-------------------------|
| uniform       | `bounded_do_while_rule` |
| asrt          | `assert_rule`           |
| asrtL         | `assert_rule_left`      |

Finally the "bwhile" rule is proven as `bounded_do_while_rule` in
`theories/Crypt/rules/RulesStateProb.v`.

We will now list the different lemmata and theorem proven on our probabilistic
relational program logic in the paper. They can all be found in
`theories/Crypt/package/pkg_rhl.v`.

**Lemma 1**
```coq
Lemma TriangleInequality :
  ∀ {Game_export : Interface} {F G H : Game_Type Game_export}
    {ϵ1 ϵ2 ϵ3}
    F ≈[ ϵ1 ] G →
    G ≈[ ϵ2 ] H →
    F ≈[ ϵ3 ] H →
    ∀ A H1 H2 H3 H4 H5 H6,
      ϵ3 A H1 H2 ≤ ϵ1 A H3 H4 + ϵ2 A H5 H6.
```

**Lemma 2**
```coq
Lemma ReductionLem :
  ∀ {Game_export M_export : Interface} {M : package Game_export M_export}
    (G : GamePair Game_export)
    (Hdisjoint0 : fdisjoint M.π1 (G false).π1)
    (Hdisjoint1 : fdisjoint M.π1 (G true).π1),
    link M (G false)
    ≈[ λ A H1 H2, @AdvantageE Game_export (G false) (G true) (link A M) (auxReduction Hdisjoint0 H1) (auxReduction Hdisjoint1 H2) ]
    link M (G true).
```

**Theorem 1**
```coq
Lemma prove_relational :
  ∀ {L₀ L₁ LA E} (p₀ p₁ : raw_package) (I : precond) (A : raw_package)
    `{ValidPackage L₀ Game_import E p₀}
    `{ValidPackage L₁ Game_import E p₁}
    `{ValidPackage LA E A_export A},
    INV' L₀ L₁ I →
    I (empty_heap, empty_heap) →
    fdisjoint LA L₀ →
    fdisjoint LA L₁ →
    eq_up_to_inv E I p₀ p₁ →
    AdvantageE p₀ p₁ A = 0.
```

**Theorem 2**
```coq
Lemma Pr_eq_empty :
  ∀ {X Y : ord_choiceType}
    {A : pred (X * heap_choiceType)} {B : pred (Y * heap_choiceType)}
    Ψ ϕ
    (c1 : FrStP heap_choiceType X) (c2 : FrStP heap_choiceType Y)
    ⊨ ⦃ Ψ ⦄ c1 ≈ c2 ⦃ ϕ ⦄ →
    Ψ (empty_heap, empty_heap) →
    (∀ x y,  ϕ x y → (A x) ↔ (B y)) →
    \P_[ θ_dens (θ0 c1 empty_heap) ] A =
    \P_[ θ_dens (θ0 c2 empty_heap) ] B.
```

### Semantic model and soundness of rules

This part of the mapping corresponds to section 5.

#### 5.1 Relational effect observation

In `theories/Relational/OrderEnrichedCategory.v` we introduce some abstract
notions such as categories, functors, relative monads, lax morphisms of relative
monads and isomorphisms of functors, all of which are order-enriched.
The file `theories/Relational/OrderEnrichedCategory.v` instantiate all of these
abstract notions.

Free monads are defined in
`theories/Crypt/rhl_semantics/free_monad/FreeProbProg.v`.

In `theories/Crypt/rhl_semantics/ChoiceAsOrd.v` we introduce the category of
choice types (`choiceType`) which are useful for sub-distributions:
they are basically the types from which we can sample.
They are one of the reason why our monads are always relative.

More basic categories can be found in the directory
`theories/Crypt/rhl_semantics/more_categories/`, namely in the files
`RelativeMonadMorph_prod.v`, `LaxComp.v`, `LaxFunctorsAndTransf.v` and
`InitialRelativeMonad.v`.


#### 5.2 The probabilistic relational effect observation

The theory for §5.2 is developed in the following files:
`theories/Crypt/rhl_semantics/only_prob/Couplings.v`
`theories/Crypt/rhl_semantics/only_prob/Theta_dens.v`
`(theories/Crypt/rhl_semantics/only_prob/Theta_exCP.v)`
`theories/Crypt/rhl_semantics/only_prob/ThetaDex.v`


#### 5.3 The stateful and probabilistic relational effect observation

The theory for §5.3 is developed in the following files, divided in two
lists, one for abstract results, and one for the instances we use.

Abstract (in `theories/Crypt/rhl_semantics/more_categories/`):
`OrderEnrichedRelativeAdjunctions.v`
`LaxMorphismOfRelAdjunctions.v`
`TransformingLaxMorph.v`

Instances (in `theories/Crypt/rhl_semantics/state_prob/`):
`OrderEnrichedRelativeAdjunctionsExamples.v`
`StateTransformingLaxMorph.v`
`StateTransfThetaDens.v`
`LiftStateful.v`


## Axioms and assumptions

### Axioms

Throughout the development we rely on the axioms of functional extensionality,
proof irrelevance, as well as propositional extensionality as listed below:

```coq
ax_proof_irrel : ClassicalFacts.proof_irrelevance
propositional_extensionality : ∀ P Q : Prop, P ↔ Q → P = Q
functional_extensionality_dep :
  ∀ (A : Type) (B : A → Type) (f g : ∀ x : A, B x),
      (∀ x : A, f x = g x) → f = g
```

We further rely on the existence of a type of real numbers as used in the
`mathcomp` library.

```coq
R : realType
```

By using `mathcomp-analysis` we also inherit one of the admitted lemmata they
have:

```coq
interchange_psum :
  ∀ (R : realType) (T U : choiceType) (S : T → U → R),
    (∀ x : T, summable (T:=U) (R:=R) (S x)) →
    summable (T:=T) (R:=R) (λ x : T, psum (λ y : U, S x y)) →
    psum (λ x : T, psum (λ y : U, S x y)) =
    psum (λ y : U, psum (λ x : T, S x y))
```

### Admitted lemma

Our development currently still contains a few unproven results. All the results
presented in the paper are formalised without admitted dependencies (besides
those announced above) except for one admitted lemma in the ElGamal example:

```coq
Lemma UniformIprod_UniformUniform :
  ∀ (i j : Index),
    ⊢ ⦃ λ '(s₀, s₁), s₀ = s₁ ⦄
      xy ← sample U (i_prod i j) ;; ret xy ≈
      x ← sample U i ;; y ← sample U j ;; ret (x, y)
    ⦃ eq ⦄.
```

It states that sampling from the uniform distribution over a product is the same
as sampling over two uniform distributions.

### Methodology for finding axioms

Our methodology to find such dependencies is to use the `Print Assumptions`
command of Coq which lists axioms a definition depends on.
For instance
```coq
Print Assumptions par_commut.
```
will yield
```coq
Axioms:
π.rel_choiceTypes : Type
boolp.propositional_extensionality : ∀ P Q : Prop, P ↔ Q → P = Q
π.probE : Type → Type
boolp.functional_extensionality_dep
  : ∀ (A : Type) (B : A → Type) (f g : ∀ x : A, B x),
	  (∀ x : A, f x = g x) → f = g
π.chEmb : π.rel_choiceTypes → choiceType
```

Note that `π.rel_choiceTypes`, `π.probE` and `π.chEmb` are not actually axioms
but instead parameters of the `Package` module, which are listed nonetheless.
