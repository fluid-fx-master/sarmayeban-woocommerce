# Quality evidence — Sarmayeban 1.0.1

Every number on this page comes from a real run of the project's own automated checks, on the date given.
The source code is private, so the scripts themselves are not published here; each section names the check that
produced the number. All runs used a private, local test WordPress with fictional data; nothing ran on a real store.

## At a glance

| What | Result | Source |
|---|---|---|
| Automated tests | **165 tests per configuration, 4 configurations, all passing** (plus 5 tests with WooCommerce inactive) | PHPUnit inside the release build, 2026-10-13 |
| Version matrix | PHP 7.4 + WordPress 6.7.9 + WooCommerce 10.3.0 and PHP 8.3 + WordPress 7.1.3 + WooCommerce 11.2.0, each with HPOS on and off | same |
| Static analysis | PHPCS (WordPress Coding Standards, PHP 7.4 compatibility), PHPStan, WordPress.org Plugin Check: no errors | same |
| Large store | 10,000 products and 50,000 orders: slowest request 11.56 s (limit 20 s), calculation memory 16.1 MB, heaviest page 11.7 MB | performance test, 2026-10-13 |
| Shared hosting | 20,000 products and 100,000 orders under a hard 128 MB memory cap, 30 s time limit and 4 MB MySQL packet: calculation, every page and the CSV complete, 0 PHP errors | shared-hosting test, 2026-10-13 |
| Security | 0 critical or high findings in two automated reviews; 30 real-HTTP attack checks passing | security review and HTTP attack test, 2026-10-12 / 2026-10-13 |
| Coexistence | 30 of 30 checks identical with and without the two most-installed Jalali-date plugins | coexistence test, 2026-10-13 |
| Install to first number | 121 seconds | visual acceptance test |

## 1. Correctness: tests against known answers

- Before any code was written, four fictional test stores were specified with **hand-calculated answer tables**
  (capital tied up, dead stock, days left, purchase suggestions, fast/slow movers). A seeding script builds exactly
  those stores, and the tests compare every number with the tables.
- Example from the reference store: recorded capital 25,000,000 and estimated capital 2,000,000 (kept apart, never
  added), 12% of recorded capital in dead stock, 2 products running out, suggested purchases of 15, 16 and 12 units.
- 20 test files, 5,578 lines of test code (line count of the test folder, 2026-10-14).
- Required kinds of tests, all present: unit (velocity, days of cover, reorder point, dead-stock groups, ABC,
  estimated cost, Jalali conversion), full calculation on each store with HPOS on and off, store modes (no costs,
  no stock management, mixed), access, and the proofs below.
- A batched calculation (many small requests) is tested to give exactly the same numbers as a single pass.

## 2. Proofs of the product's promises

| Promise | How it is proven |
|---|---|
| Read-only: never changes prices, stock, orders, products or costs | A test snapshots all store data before and after every page, the calculation and the CSV download, and compares them |
| No outgoing requests | A test makes any outgoing HTTP request fail; the plugin ships all its CSS and JS |
| Access control | Subscribers and visitors are refused on every page, action and download; shop managers allowed (unit tests and real HTTP) |
| No buyer personal data | Only product-level totals are stored; log messages are stripped of e-mail addresses and long numbers |
| Clean uninstall | Real activation, deactivation and uninstall on a fresh WordPress: only the plugin's own data is removed, and only when the owner allowed it |

## 3. Performance

Store of 10,000 products and 50,000 orders, measured request by request (performance test, 2026-10-13, version 1.0.1):

| | WooCommerce 11.2 / PHP 8.3 | WooCommerce 10.3 / PHP 7.4 | Limit |
|---|---|---|---|
| Requests / total time | 45 / about 7 minutes | 38 / about 6 minutes | — |
| Slowest request | 11.56 s | 11.89 s | 20 s |
| Calculation memory (extra) | 16.1 MB | not measurable on PHP < 8.2 | 64 MB |
| Heaviest result page | 11.7 MB | not measurable on PHP < 8.2 (every page under 0.5 s) | 128 MB |
| CSV download | 6.9 MB | — | 128 MB |

Before the hosting fixes of version 1.0.1 the same test measured 30 MB for the calculation and 74.6 MB for the
heaviest page.

## 4. Shared-hosting robustness

Simulated shared host (hard 128 MB PHP memory cap, 30 s time limit, 4 MB MySQL `max_allowed_packet`), store of
20,000 products and 100,000 orders, every request over real HTTP (shared-hosting test, 2026-10-13):

| | Version 1.0.0 | Version 1.0.1 |
|---|---|---|
| Calculation | did not start (memory error) | complete: 80 requests, about 15 minutes, slowest 12.7 s |
| Result pages | 4 of 5 failed (memory error) | all open, 107–109 MB peak (most of it WordPress and WooCommerce themselves) |
| PHP errors | 4 | 0 |

What changed in 1.0.1: rows are read in chunks of 500, long tables show 100 rows a page, orders are read one day at
a time instead of listing every order ID, and a fatal error releases the calculation lock at once. Each fix has its
own test, and each test was checked by breaking the code on purpose (it failed, then passed again).

## 5. Security

- **Automated review 1** (three parallel reviewers: access and CSRF, output and CSV, data and robustness):
  0 critical or high; 4 low security findings and 6 robustness findings fixed, each with a test.
- **Automated review 2** (line by line, plus attacks over real HTTP): no new findings. The attack test now has
  30 checks: logged-out visitor, subscriber, missing or wrong nonce, GET on POST-only actions, direct requests to
  every PHP file, reflected XSS in every URL argument (including the page links added in 1.0.1), open redirects,
  80 malformed form posts, CSV and frame headers, and the PHP log.
- **Independent red-team review** during development: 13 issues fixed, 4 accepted with written reasons.
- CSV formula injection is neutralised (cells starting with `=`, `+`, `-`, `@`, also after spaces or invisible marks).
- **Not yet done:** a review by a human security specialist. It is required before the first sale, and a handoff
  package for that reviewer is ready.

## 6. Compatibility

- Both WooCommerce order storages (HPOS and legacy), in every test configuration.
- Alongside the two most-installed Jalali-date plugins on WordPress.org (Parsi Date 6.4 and Persian WooCommerce
  10.0.5, 100,000 active installs each): five setups × the 90-day window and a custom Jalali range × both order
  storages; pages, dates and CSV byte for byte identical to the run without them, no date converted twice, no PHP
  error. The test was checked by routing the plugin's dates through the filter those plugins change: it caught it
  64 times.

## 7. Release process

One command builds the release: it runs every check above, packages only the shipped files from the committed code,
checks the package for development files, installs it on a brand-new WordPress + WooCommerce (oldest and newest
supported versions) where it activates the plugin, opens every page over HTTP and deletes it with and without its
data, runs the WordPress.org Plugin Check on the package, and only then tags the release.

## Not tested yet

- A real hosting provider (every test ran on a local machine).
- Persistent object caches (Redis, Memcached) and very large multisite networks.
- Multi-currency plugins.
