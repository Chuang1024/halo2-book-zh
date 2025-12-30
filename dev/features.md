# Feature development

Sometimes feature development can require iterating on a design over time. It can be
useful to start using features in downstream crates early on to gain experience with the
APIs and functionality, that can feed back into the feature's design prior to it being
stabilised. To enable this, we follow a three-stage `nightly -> beta -> stable`
development pattern inspired by (but not identical to) the Rust compiler.

## Feature flags

Each unstabilised feature has a default-off feature flag that enables it, of the form
`unstable-*`. The stable API of the crates must not be affected when the feature flag is
disabled, except for specific complex features that will be considered on a case-by-case
basis.

Two meta-flags are provided to enable all features at a particular stabilisation level:
- `beta` enables all features at the "beta" stage (and implicitly all features at the
  "stable" stage).
- `nightly` enables all features at the "beta" and "nightly" stages (and implicitly all
  features at the "stable" stage), i.e. all features are enabled.
- When neither flag is enabled (and no feature-specific flags are enabled), then in effect
  only features at the "stable" stage are enabled.

## Feature workflow

- If the maintainers have rough consensus that an experimental feature is generally
  desired, its initial implementation can be merged into the codebase optimistically
  behind a feature-specific feature flag with a lower standard of review. The feature's
  flag is added to the `nightly` feature flag set.
  - The feature will become usable by downstream published crates in the next general
    release of the `halo2` crates.
  - Subsequent development and refinement of the feature can be performed in-situ via
    additional PRs, along with additional review.
  - If the feature ends up having bad interactions with other features (in particular,
    already-stabilised features), then it can be removed later without affecting the
    stable or beta APIs.
- Once the feature has had sufficient review, and is at the point where a `halo2` user
  considers it production-ready (and is willing or planning to deploy it to production),
  the feature's feature flag is moved to the `beta` feature flag set.
- Once the feature has had review equivalent to the stable review policy, and there is
  rough consensus that the feature is useful to the wider `halo2` userbase, the feature's
  feature flag is removed and the feature becomes part of the main maintained codebase.

> For more complex features, the above workflow might be augmented with `beta` and
> `nightly` branches; this will be figured out once a feature requiring this is proposed
> as a candidate for inclusion.

## In-progress features

| Feature flag | Stage | Notes |
| --- | --- | --- |
| `unstable-sha256-gadget` | `nightly` | The SHA-256 gadget and chip.


# 功能开发

有时功能开发可能需要随着时间的推移对设计进行迭代。在功能稳定之前，早期在下游 crate 中使用这些功能可以帮助积累 API 和功能的使用经验，从而反馈到功能的设计中。为了实现这一点，我们遵循了一个三阶段的 `nightly -> beta -> stable` 开发模式，灵感来自（但不完全相同）Rust 编译器。

## 功能标志

每个未稳定的功能都有一个默认关闭的功能标志，形式为 `unstable-*`。当功能标志被禁用时，crate 的稳定 API 必须不受影响，除非是某些复杂的特性，这些特性将根据具体情况考虑。

提供了两个元标志来启用特定稳定化阶段的所有功能：
- `beta` 启用所有处于“beta”阶段的功能（并隐式启用所有处于“stable”阶段的功能）。
- `nightly` 启用所有处于“beta”和“nightly”阶段的功能（并隐式启用所有处于“stable”阶段的功能），即启用所有功能。
- 当这两个标志都未启用（且未启用任何特定功能标志）时，实际上只有处于“stable”阶段的功能被启用。

## 功能工作流程

- 如果维护者大致达成共识，认为某个实验性功能是普遍需要的，其初始实现可以在较低审查标准的情况下，通过特定功能标志乐观地合并到代码库中。该功能的标志被添加到 `nightly` 功能标志集中。
  - 该功能将在 `halo2` crate 的下一个通用版本中可供下游发布的 crate 使用。
  - 后续的功能开发和改进可以通过额外的 PR 在原位进行，同时进行额外的审查。
  - 如果该功能最终与其他功能（特别是已经稳定的功能）存在不良交互，则可以在不影响稳定或 beta API 的情况下稍后移除它。
- 一旦该功能经过充分审查，并且达到 `halo2` 用户认为其生产就绪（并愿意或计划将其部署到生产环境）的阶段，该功能的功能标志将被移动到 `beta` 功能标志集中。
- 一旦该功能经过与稳定审查政策相当的审查，并且大致达成共识认为该功能对更广泛的 `halo2` 用户群有用，该功能的功能标志将被移除，该功能将成为主维护代码库的一部分。

> 对于更复杂的功能，上述工作流程可能会通过 `beta` 和 `nightly` 分支进行增强；这将在提出需要此功能的功能作为候选时确定。

## 进行中的功能

| 功能标志 | 阶段 | 备注 |
| --- | --- | --- |
| `unstable-sha256-gadget` | `nightly` | SHA-256 小工具和芯片。
