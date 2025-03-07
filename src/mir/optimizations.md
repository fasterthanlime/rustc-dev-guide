# MIR optimizations

MIR optimizations are optimizations run on the [MIR][mir] to produce better MIR
before codegen. This is important for two reasons: first, it makes the final
generated executable code better, and second, it means that LLVM has less work
to do, so compilation is faster. Note that since MIR is generic (not
[monomorphized][monomorph] yet), these optimizations are particularly
effective; we can optimize the generic version, so all of the monomorphizations
are cheaper!

[mir]: /mir/index.md
[monomorph]: /appendix/glossary.md#mono

MIR optimizations run after borrow checking. We run a series of optimization
passes over the MIR to improve it. Some passes are required to run on all code,
some passes don't actually do optimizations but only check stuff, and some
passes are only turned on in `release` mode.

The [`optimized_mir`][optmir] [query] is called to produce the optimized MIR
for a given [`DefId`][defid]. This query makes sure that the borrow checker has
run and that some validation has occurred. Then, it [steals][steal] the MIR,
optimizes it, and returns the improved MIR.

[optmir]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_transform/fn.optimized_mir.html
[query]: /query.md
[defid]: /appendix/glossary.md#def-id
[steal]: /mir/passes.md#stealing

## Quickstart for adding a new optimization

1. Make a Rust source file in `src/test/mir-opt` that shows the code you want to
   optimize. This should be kept simple, so avoid `println!` or other formatting
   code if it's not necessary for the optimization. The reason for this is that
   `println!`, `format!`, etc. generate a lot of MIR that can make it harder to
   understand what the optimization does to the test.

2. Run `./x.py test --bless src/test/mir-opt/<your-test>.rs` to generate a MIR
   dump. Read [this README][mir-opt-test-readme] for instructions on how to dump
   things.

3. Commit the current working directory state. The reason you should commit the
   test output before you implement the optimization is so that you (and your
   reviewers) can see a before/after diff of what the optimization changed.

4. Implement a new optimization in [`compiler/rustc_mir_transform/src`].
   The fastest and easiest way to do this is to

   1. pick a small optimization (such as [`no_landing_pads`]) and copy it
      to a new file,
   2. add your optimization to one of the lists in the
      [`run_optimization_passes()`] function,
   3. and then start modifying the copied optimization.

5. Rerun `./x.py test --bless src/test/mir-opt/<your-test>.rs` to regenerate the
   MIR dumps. Look at the diffs to see if they are what you expect.

6. Run `./x.py test src/test/ui` to see if your optimization broke anything.

7. If there are issues with your optimization, experiment with it a bit and
   repeat steps 5 and 6.

8. Commit and open a PR. You can do this at any point, even if things aren't
   working yet, so that you can ask for feedback on the PR. Open a "WIP" PR
   (just prefix your PR title with `[WIP]` or otherwise note that it is a
   work in progress) in that case.

   Make sure to commit the blessed test output as well! It's necessary for CI to
   pass and it's very helpful to reviewers.

If you have any questions along the way, feel free to ask in
`#t-compiler/wg-mir-opt` on Zulip.

[mir-opt-test-readme]: https://github.com/rust-lang/rust/blob/master/src/test/mir-opt/README.md
[`compiler/rustc_mir_transform/src`]: https://github.com/rust-lang/rust/tree/master/compiler/rustc_mir_transform/src
<!--- TODO: Change NoLandingPads. [#1232](https://github.com/rust-lang/rustc-dev-guide/issues/1232) -->
[`no_landing_pads`]: https://github.com/rust-lang/rust/blob/master/compiler/rustc_mir_transform/src/no_landing_pads.rs
[`run_optimization_passes()`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_transform/fn.run_optimization_passes.html

## Defining optimization passes

The list of passes run and the order in which they are run is defined by the
[`run_optimization_passes`][rop] function. It contains an array of passes to
run.  Each pass in the array is a struct that implements the [`MirPass`] trait.
The array is an array of `&dyn MirPass` trait objects. Typically, a pass is
implemented in its own module of the [`rustc_mir_transform`][trans] crate.

[rop]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_transform/fn.run_optimization_passes.html
[`MirPass`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/trait.MirPass.html
[trans]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_transform/index.html

Some examples of passes are:
- `CleanupNonCodegenStatements`: remove some of the info that is only needed for
  analyses, rather than codegen.
- `ConstProp`: Does [constant propagation][constprop]

You can see the ["Implementors" section of the `MirPass` rustdocs][impl] for more examples.

[impl]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/trait.MirPass.html#implementors
[constprop]: https://en.wikipedia.org/wiki/Constant_folding#Constant_propagation

## MIR optimization levels

MIR optimizations can come in various levels of readiness. Experimental
optimizations may cause miscompilations, or slow down compile times.
These passes are still included in nightly builds to gather feedback and make it easier to modify
the pass. To enable working with slow or otherwise experimental optimization passes,
you can specify the `-Z mir-opt-level` debug flag. You can find the
definitions of the levels in the [compiler MCP]. If you are developing a MIR pass and
want to query whether your optimization pass should run, you can check the
current level using [`tcx.sess.opts.unstable_opts.mir_opt_level`][mir_opt_level].

[compiler MCP]: https://github.com/rust-lang/compiler-team/issues/319
[mir_opt_level]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_session/config/struct.UnstableOptions.html#structfield.mir_opt_level

## Optimization fuel

Optimization fuel is a compiler option (`-Z fuel=<crate>=<value>`) that allows for fine grained
control over which optimizations can be applied during compilation: each optimization reduces
fuel by 1, and when fuel reaches 0 no more optimizations are applied. The primary use of fuel
is debugging optimizations that may be incorrect or misapplied. By changing the fuel
value, you can bisect a compilation session down to the exact incorrect optimization
(this behaves like a kind of binary search through the optimizations).

MIR optimizations respect fuel, and in general each pass should check fuel by calling
[`tcx.consider_optimizing`][consideroptimizing] and skipping the optimization if fuel
is empty. There are a few considerations:

1. If the pass is considered "guaranteed" (for example, it should always be run because it is
needed for correctness), then fuel should not be used. An example of this is `PromoteTemps`.
2. In some cases, an initial pass is performed to gather candidates, which are then iterated to
perform optimizations. In these situations, we should allow for the initial gathering pass
and then check fuel as close to the mutation as possible. This allows for the best
debugging experience, because we can determine where in the list of candidates an optimization
may have been misapplied. Examples of this are `InstCombine` and `ConstantPropagation`.

[consideroptimizing]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/context/struct.TyCtxt.html#method.consider_optimizing
