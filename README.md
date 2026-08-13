# CIB seven Community Helm Chart

Community Helm charts for [CIB seven](https://github.com/cibseven), the CIB fork of the Camunda 7
process engine.

This repo was forked from [camunda-community-hub/camunda-helm](https://github.com/camunda-community-hub/camunda-helm)
and adapted to install the CIB seven distribution by default.

## Charts

* [cibseven](./charts/cibseven) — CIB seven engine (Cockpit / Tasklist / Admin).

## CI/CD

The CI/CD are done in GitHub Actions, and main actions are used:

* Testing charts via Helm [chart-testing-action](https://github.com/helm/chart-testing-action).
* Validating charts with different Kubernetes versions via [kind-action](https://github.com/helm/kind-action).
* Releasing charts via Helm [chart-releaser-action](https://github.com/helm/chart-releaser-action).

## Project status

This project is under active development, and breaking changes may occur — it follows SemVer.

## Contributing

We value all feedback and contributions. If you find any issues or want to contribute, please
open an issue or PR against this repository.

## License

This is open source software licensed using the Apache License 2.0. Please see [LICENSE](LICENSE) for details.
