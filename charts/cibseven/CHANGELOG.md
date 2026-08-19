The changelog is automatically generated using [git-chglog](https://github.com/git-chglog/git-chglog) and it follows [Keep a Changelog](https://keepachangelog.com) format.

<a name="cibseven-0.0.4"></a>
## [cibseven-0.0.4] - 2026-08-19
### Added
- Added helm repo add instructions to chart README

<a name="cibseven-0.0.3"></a>
## [cibseven-0.0.3] - 2026-08-18
### Changed
- Changed webclient values section to distro (Spring Boot application.yaml override applies to whole CIB seven distribution, not only the webclient webapp)
### BREAKING CHANGE

rename `webclient.applicationYaml` to `distro.applicationYaml` in your values files. The rendered ConfigMap and volume name also change from `<fullname>-webclient-config` to `<fullname>-distro-config`.


<a name="cibseven-0.0.2"></a>
## [cibseven-0.0.2] - 2026-08-13

<a name="cibseven-0.0.1"></a>
## cibseven-0.0.1 - 2026-08-11
### Added
- Added webclient.applicationYaml Spring Boot config override
### Changed
- Changed chart version to 1.0.0
- Changed default chart to CIB seven (cibseven/cibseven engine)

[cibseven-0.0.4]: https://github.com/cibseven-community-hub/cibseven-community-helm-chart/compare/cibseven-0.0.3...cibseven-0.0.4
[cibseven-0.0.3]: https://github.com/cibseven-community-hub/cibseven-community-helm-chart/compare/cibseven-0.0.2...cibseven-0.0.3
[cibseven-0.0.2]: https://github.com/cibseven-community-hub/cibseven-community-helm-chart/compare/cibseven-0.0.1...cibseven-0.0.2
