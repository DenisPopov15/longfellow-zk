## Local ZK Experiments

- **One-time setup (dependencies for tests and benchmarks)**

  These are needed because CMake’s `CMake/proofs.cmake` uses `find_package(benchmark)` and `find_package(GTest)`:

  ```bash
  # Install Google Benchmark (used by various *_benchmark targets)
  brew install google-benchmark

  # Install GoogleTest so CMake can find GTest for unit tests
  brew install googletest
  ```

- **Configure the Release build (creates `clang-build-release/`)**

  Mirrors the instructions in `README.md`, but targets a `clang-build-release` build directory:

  ```bash
  cd /path/to/longfellow-zk   # repo root

  CXX=clang++ cmake -D CMAKE_BUILD_TYPE=Release \
    -S lib -B clang-build-release \
    --install-prefix ${PWD}/install
  ```

- **Build everything (including the JWT test binary)**

  This compiles all tests and tools, including `circuits/tests/jwt/jwt_test`:

  ```bash
  cd clang-build-release
  make -j16
  ```

- **Run only the new JWT single-attribute ZK test**

  This runs the end-to-end ZK test that exercises the JWT circuit with one opened attribute:

  ```bash
  cd clang-build-release
  ctest -R JwtSingleAttributeZk11 -V
  # which internally invokes:
  # ./circuits/tests/jwt/jwt_test --gtest_filter=jwt.JwtSingleAttributeZk11
  ```

or 
```
cd clang-build-release
  make -j16 circuits/tests/jwt/jwt_test
  ./circuits/tests/jwt/jwt_test --gtest_filter=jwt.JwtSingleAttributeZk11
```