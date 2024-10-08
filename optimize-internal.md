 ## optimize核心源码入口：
```java
  /**
   * Optimizes a query plan.
   *
   * @param root Root of relational expression tree
   * @param materializations Tables known to be populated with a given query
   * @param lattices Lattices
   * @return an equivalent optimized relational expression
   */
  protected RelRoot optimize(RelRoot root,
      final List<Materialization> materializations,
      final List<CalciteSchema.LatticeEntry> lattices) {
    final RelOptPlanner planner = root.rel.getCluster().getPlanner();

    final DataContext dataContext = context.getDataContext();
    planner.setExecutor(new RexExecutorImpl(dataContext));

    final List<RelOptMaterialization> materializationList =
        new ArrayList<>(materializations.size());
    for (Materialization materialization : materializations) {
      List<String> qualifiedTableName = materialization.materializedTable.path();
      materializationList.add(
          new RelOptMaterialization(
              castNonNull(materialization.tableRel),
              castNonNull(materialization.queryRel),
              materialization.starRelOptTable,
              qualifiedTableName));
    }

    final List<RelOptLattice> latticeList = new ArrayList<>(lattices.size());
    for (CalciteSchema.LatticeEntry lattice : lattices) {
      final CalciteSchema.TableEntry starTable = lattice.getStarTable();
      final JavaTypeFactory typeFactory = context.getTypeFactory();
      final RelOptTableImpl starRelOptTable =
          RelOptTableImpl.create(catalogReader,
              starTable.getTable().getRowType(typeFactory), starTable, null);
      latticeList.add(
          new RelOptLattice(lattice.getLattice(), starRelOptTable));
    }

    final RelTraitSet desiredTraits = getDesiredRootTraitSet(root);

    final Program program = getProgram();
    final RelNode rootRel4 =
        program.run(planner, root.rel, desiredTraits, materializationList,
            latticeList);
    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("Plan after physical tweaks:\n{}",
          RelOptUtil.toString(rootRel4, SqlExplainLevel.ALL_ATTRIBUTES));
    }

    return root.withRel(rootRel4);
  }
```

## 主要逻辑在 final Program program = getProgram();

```java
protected Program getProgram() {
    // Allow a test to override the default program.
    final Holder<@Nullable Program> holder = Holder.empty();
    Hook.PROGRAM.run(holder);
    @Nullable Program holderValue = holder.get();
    if (holderValue != null) {
      return holderValue;
    }

    return Programs.standard();
  }

 /** Returns the standard program used by Prepare. */
  public static Program standard() {
    return standard(DefaultRelMetadataProvider.INSTANCE, true);
  }

 /** Returns the standard program with user metadata provider and enableFieldTrimming config. */
  public static Program standard(RelMetadataProvider metadataProvider,
      boolean enableFieldTrimming) {
    final Program program1 =
        (planner, rel, requiredOutputTraits, materializations, lattices) -> {
          for (RelOptMaterialization materialization : materializations) {
            planner.addMaterialization(materialization);
          }
          for (RelOptLattice lattice : lattices) {
            planner.addLattice(lattice);
          }

          planner.setRoot(rel);
          final RelNode rootRel2 =
              rel.getTraitSet().equals(requiredOutputTraits)
                  ? rel
                  : planner.changeTraits(rel, requiredOutputTraits);
          assert rootRel2 != null;

          planner.setRoot(rootRel2);
          final RelOptPlanner planner2 = planner.chooseDelegate();
          final RelNode rootRel3 = planner2.findBestExp();
          assert rootRel3 != null : "could not implement exp";
          return rootRel3;
        };

    List<Program> programs =
        Lists.newArrayList(subQuery(metadataProvider),
        new DecorrelateProgram(),
        measure(metadataProvider),
        new TrimFieldsProgram(),
        program1,

        // Second planner pass to do physical "tweaks". This the first time
        // that EnumerableCalcRel is introduced.
        calc(metadataProvider));

    programs.removeIf(program -> !enableFieldTrimming && program instanceof TrimFieldsProgram);

    return new SequenceProgram(ImmutableList.copyOf(programs));
  }
```

## 接下来会调用VolcanoPlanner的planner.setRoot(rel);
```java
@Override public void setRoot(RelNode rel) {
    this.root = registerImpl(rel, null);
    if (this.originalRoot == null) {
      this.originalRoot = rel;
    }

    rootConvention = this.root.getConvention();
    ensureRootConverters();
  }

/**
   * Registers a new expression <code>exp</code> and queues up rule matches.
   * If <code>set</code> is not null, makes the expression part of that
   * equivalence set. If an identical expression is already registered, we
   * don't need to register this one and nor should we queue up rule matches.
   *
   * @param rel relational expression to register. Must be either a
   *         {@link RelSubset}, or an unregistered {@link RelNode}
   * @param set set that rel belongs to, or <code>null</code>
   * @return the equivalence-set
   */
  private RelSubset registerImpl(
      RelNode rel,
      @Nullable RelSet set) {
    if (rel instanceof RelSubset) {
      return registerSubset(set, (RelSubset) rel);
    }

    assert !isRegistered(rel) : "already been registered: " + rel;
    if (rel.getCluster().getPlanner() != this) {
      throw new AssertionError("Relational expression " + rel
          + " belongs to a different planner than is currently being used.");
    }

    // Now is a good time to ensure that the relational expression
    // implements the interface required by its calling convention.
    final RelTraitSet traits = rel.getTraitSet();
    final Convention convention = traits.getTrait(ConventionTraitDef.INSTANCE);
    assert convention != null;
    if (!convention.getInterface().isInstance(rel)
        && !(rel instanceof Converter)) {
      throw new AssertionError("Relational expression " + rel
          + " has calling-convention " + convention
          + " but does not implement the required interface '"
          + convention.getInterface() + "' of that convention");
    }
    if (traits.size() != traitDefs.size()) {
      throw new AssertionError("Relational expression " + rel
          + " does not have the correct number of traits: " + traits.size()
          + " != " + traitDefs.size());
    }

    // Ensure that its sub-expressions are registered.
    rel = rel.onRegister(this);

    // Record its provenance. (Rule call may be null.)
    final VolcanoRuleCall ruleCall = ruleCallStack.peek();
    if (ruleCall == null) {
      provenanceMap.put(rel, Provenance.EMPTY);
    } else {
      provenanceMap.put(
          rel,
          new RuleProvenance(
              ruleCall.rule,
              ImmutableList.copyOf(ruleCall.rels),
              ruleCall.id));
    }

    // If it is equivalent to an existing expression, return the set that
    // the equivalent expression belongs to.
    RelDigest digest = rel.getRelDigest();
    RelNode equivExp = mapDigestToRel.get(digest);
    if (equivExp == null) {
      // do nothing
    } else if (equivExp == rel) {
      // The same rel is already registered, so return its subset
      return getSubsetNonNull(equivExp);
    } else {
      if (!RelOptUtil.areRowTypesEqual(equivExp.getRowType(),
          rel.getRowType(), false)) {
        throw new IllegalArgumentException(
            RelOptUtil.getFullTypeDifferenceString("equiv rowtype",
                equivExp.getRowType(), "rel rowtype", rel.getRowType()));
      }
      checkPruned(equivExp, rel);

      RelSet equivSet = getSet(equivExp);
      if (equivSet != null) {
        LOGGER.trace(
            "Register: rel#{} is equivalent to {}", rel.getId(), equivExp);
        return registerSubset(set, getSubsetNonNull(equivExp));
      }
    }

    // Converters are in the same set as their children.
    if (rel instanceof Converter) {
      final RelNode input = ((Converter) rel).getInput();
      final RelSet childSet = castNonNull(getSet(input));
      if ((set != null)
          && (set != childSet)
          && (set.equivalentSet == null)) {
        LOGGER.trace(
            "Register #{} {} (and merge sets, because it is a conversion)",
            rel.getId(), rel.getRelDigest());
        merge(set, childSet);

        // During the mergers, the child set may have changed, and since
        // we're not registered yet, we won't have been informed. So
        // check whether we are now equivalent to an existing
        // expression.
        if (fixUpInputs(rel)) {
          digest = rel.getRelDigest();
          RelNode equivRel = mapDigestToRel.get(digest);
          if ((equivRel != rel) && (equivRel != null)) {

            // make sure this bad rel didn't get into the
            // set in any way (fixupInputs will do this but it
            // doesn't know if it should so it does it anyway)
            set.obliterateRelNode(rel);

            // There is already an equivalent expression. Use that
            // one, and forget about this one.
            return getSubsetNonNull(equivRel);
          }
        }
      } else {
        set = childSet;
      }
    }

    // Place the expression in the appropriate equivalence set.
    if (set == null) {
      set =
          new RelSet(nextSetId++,
              Util.minus(RelOptUtil.getVariablesSet(rel),
                  rel.getVariablesSet()),
              RelOptUtil.getVariablesUsed(rel));
      this.allSets.add(set);
    }
```

## 先深度优先注册子节点
```java
@Override public RelNode onRegister(RelOptPlanner planner) {
    List<RelNode> oldInputs = getInputs();
    List<RelNode> inputs = new ArrayList<>(oldInputs.size());
    for (final RelNode input : oldInputs) {
      RelNode e = planner.ensureRegistered(input, null);
      assert e == input || RelOptUtil.equal("rowtype of rel before registration",
          input.getRowType(),
          "rowtype of rel after registration",
          e.getRowType(),
          Litmus.THROW);
      inputs.add(e);
    }
    RelNode r = this;
    if (!Util.equalShallow(oldInputs, inputs)) {
      r = copy(getTraitSet(), inputs);
    }
    r.recomputeDigest();
    assert r.isValid(Litmus.THROW, null);
    return r;
  }

@Override public RelSubset ensureRegistered(RelNode rel, @Nullable RelNode equivRel) {
    RelSubset result;
    final RelSubset subset = getSubset(rel);
    if (subset != null) {
      if (equivRel != null) {
        final RelSubset equivSubset = getSubsetNonNull(equivRel);
        if (subset.set != equivSubset.set) {
          merge(equivSubset.set, subset.set);
        }
      }
      result = canonize(subset);
    } else {
      result = register(rel, equivRel);
    }

    // Checking if tree is valid considerably slows down planning
    // Only doing it if logger level is debug or finer
    if (LOGGER.isDebugEnabled()) {
      assert isValid(Litmus.THROW);
    }

    return result;
  }

 @Override public RelSubset register(
      RelNode rel,
      @Nullable RelNode equivRel) {
    assert !isRegistered(rel) : "pre: isRegistered(rel)";
    final RelSet set;
    if (equivRel == null) {
      set = null;
    } else {
      final RelDataType relType = rel.getRowType();
      final RelDataType equivRelType = equivRel.getRowType();
      if (!RelOptUtil.areRowTypesEqual(relType,
          equivRelType, false)) {
        throw new IllegalArgumentException(
            RelOptUtil.getFullTypeDifferenceString("rel rowtype", relType,
                "equiv rowtype", equivRelType));
      }
      equivRel = ensureRegistered(equivRel, null);
      set = getSet(equivRel);
    }
    return registerImpl(rel, set);
  }
```

## 然后又调用了registerImpl(rel, set) 进行深度优先的递归， 接下来看一下注册过程做了啥
主要逻辑是触发规则rule:

```java
// Queue up all rules triggered by this relexp's creation.
    fireRules(rel);

 /**
   * Fires all rules matched by a relational expression.
   *
   * @param rel      Relational expression which has just been created (or maybe
   *                 from the queue)
   */
  void fireRules(RelNode rel) {
    for (RelOptRuleOperand operand : classOperands.get(rel.getClass())) {
      if (operand.matches(rel)) {
        final VolcanoRuleCall ruleCall;
        ruleCall = new DeferringRuleCall(this, operand);
        ruleCall.match(rel);
      }
    }
  }

  /**
   * Applies this rule, with a given relational expression in the first slot.
   */
  void match(RelNode rel) {
    assert getOperand0().matches(rel) : "precondition";
    final int solve = 0;
    int operandOrdinal = castNonNull(getOperand0().solveOrder)[solve];
    this.rels[operandOrdinal] = rel;
    matchRecurse(solve + 1);
  }
```

## 接下来根据匹配的规则进行应用





