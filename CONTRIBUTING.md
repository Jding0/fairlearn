# Contributing to fairlearn

This project welcomes contributions and suggestions.

## Developer certificate of origin
Contributions require you to sign a _developer certificate of origin_ (DCO) declaring that you have the right to, and actually do, grant us the rights to use your contribution. For details, visit https://developercertificate.org/.

When you submit a pull request, a DCO-bot will automatically determine whether you need to provide a DCO and decorate the PR appropriately (e.g., label, comment). Simply follow the instructions provided by the bot. You will only need to do this once across all repositories using our DCO.

## Code of conduct
This project has adopted the [GitHub community guidelines](https://help.github.com/en/github/site-policy/github-community-guidelines).

## Development process
Development happens against the `master` branch following the [GitHub flow model](https://guides.github.com/introduction/flow/). Contributors create feature branches off of `master`, and their pull requests should target the `master` branch. Maintainers are responsible for prompt review of pull requests.

Pull requests against `master` trigger automated tests that are run through Azure DevOps. Additional test suites are run periodically. When adding new code paths or features, tests are a requirement to complete a pull request. They should be added in the `test` directory.

### Investigating automated test failures
For every pull request to `master` with automated tests, you can check the logs of the tests to find the root cause of failures. Our tests currently run through Azure Pipelines with steps for setup, testing, and teardown. The `Checks` tab of a pull request contains a link to the [Azure Pipelines page](dev.azure.com/responsibleai/fairlearn/_build/results), where you can review the logs by clicking on a specific step in the automated test sequence. If you encounter problems with this workflow, please reach out through [GitHub issues](https://github.com/fairlearn/fairlearn/issues).

To run the same tests locally, find the corresponding pipeline definition (a `yml` file) in the `devops` directory. It either directly contains the command to execute the tests (usually starting with `python -m pytest`) or it refers to a template file with the command.

## API conventions

This section relies on the definitions from our [terminology guide](TERMINOLOGY.md), including the definitions of "estimator", "reduction", "sensitive features", "moment", and "parity".

### Unfairness mitigation algorithms

Unfairness mitigation algorithms take form of scikit-learn-style estimators. Any algorithm-specific parameters are passed to the constructor. The resulting instance of the algorithm should support methods `fit` and `predict` with APIs resembling those of scikit-learn as much as possible. Any deviations are noted below.

#### Reductions

Reduction constructors require a parameter corresponding to an estimator that implements the `fit` method with the `sample_weight` argument. Parity constraints for reductions are expressed via instances of various subclasses of the class `fairlearn.reductions.Moment`. Formally, instances of the class `Moment` implement vector-valued random variables whose sample averages over the data are required to be bounded (above and/or below).

```python
constraints = Moment()
reduction = Reduction(estimator, constraints)
```

Reductions provide `fit` and `predict` methods with the following signatures:

```python
reduction.fit(X, y, **kwargs)
reduction.predict(X)
```

All of the currently supported parity constraints (subclasses of `Moment`) are based on sensitive features that need to be provided to `fit` as a keyword argument `sensitive_features`. In the future, it will also be possible to provide sensitive features as columns of `X`.

#### Post-processing algorithms

The constructors of post-processing algorithms require either an already trained predictor or an estimator (which is trained on the data when executing `fit`). For post-processing algorithms, the `constraints` argument is provided as a string.

```python
postprocessor = PostProcessing(unconstrained_predictor=predictor, constraints=constraints)
postprocessor = PostProcessing(estimator=estimator, constraints=constraints)
```

Post-processing algorithms (such as the ones under `fairlearn.postprocessing`) provide the same functions as the reductions above albeit with `sensitive_features` as a required argument for `predict`. In the future, we will make `sensitive_features` optional if the sensitive features are already provided through `X`.

```python
postprocessor.fit(X, y, sensitive_features=sensitive_features)
postprocessor.predict(X, sensitive_features=sensitive_features)
```


## Creating new releases

We have a [Azure DevOps Pipeline](https://dev.azure.com/responsibleai/fairlearn/_build?definitionId=48&_a=summary) which takes care of building wheels and pushing to PyPI. Validations are also performed prior to any deployments, and also following the uploads to Test-PyPI and PyPI. To use it:
1. Ensure that `_base_version` in `fairlearn/__init__.py` is set correctly for PyPI
1. Put down a tag corresponding to this `_base_version` but preprended with `v`. For example, version `0.5.0` should be tagged wtih `v0.5.0`
1. Queue the pipeline at this tag, with a variable `DEV_VERSION` set to zero. When the package is uploaded to Test-PyPI, this number will be appended to the version as a `dev[n]` suffix

The pipeline requires sign offs immediately prior to the deployments to Test-PyPI and PyPI. If there is an issue found, then after applying the fix, update the location of the tag, and queue a new release pipeline with the value of `DEV_VERSION` increased by one.

The `DEV_VERSION` variable is to work around the PyPI behaviour where uploads are immutable and published immediately. This means that each upload to PyPI 'uses up' that particular version (on that particular PyPI instance). Since we wish to deploy exactly the same bits to both Test-PyPI and PyPI, without this workaround the version released to PyPI would depend on the number of issues discovered when uploading to Test-PyPI. If PyPI updates their release process to separate the 'upload' and 'publish' steps (i.e. until a package is published, it remains hidden and mutable) then the code associated with `DEV_VERSION` should be removed.

As part of the release process, the `build-wheels.ps1` script uses `process_readme.py` to turn all the relative links in the ReadMe file into absolute ones (this is the reason why the applied tag has be of the form `v[_base_version]`). The `process_readme.py` script is slightly fragile with respect to the contents of the ReadMe, so after significant changes its output should be verified.