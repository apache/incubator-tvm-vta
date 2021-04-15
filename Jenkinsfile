#!groovy
// -*- mode: groovy -*-

// Licensed to the Apache Software Foundation (ASF) under one
// or more contributor license agreements.  See the NOTICE file
// distributed with this work for additional information
// regarding copyright ownership.  The ASF licenses this file
// to you under the Apache License, Version 2.0 (the
// "License"); you may not use this file except in compliance
// with the License.  You may obtain a copy of the License at
//
//   http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing,
// software distributed under the License is distributed on an
// "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
// KIND, either express or implied.  See the License for the
// specific language governing permissions and limitations
// under the License.

// Jenkins pipeline
// See documents at https://jenkins.io/doc/book/pipeline/jenkinsfile/

ci_lint = "tvmai/ci-lint:v0.51"
ci_cpu = "tvmai/ci-cpu:v0.55"
ci_i386 = "tvmai/ci-i386:v0.52"

// command to start a docker container
// NOTE: docker container provides an extra layer of isolation
docker_run = "tests/scripts/docker_bash.sh"
// timeout in minutes
max_time = 240

// initialize source codes
def init_git() {
  checkout scm
  retry(5) {
    timeout(time: 2, unit: "MINUTES") {
      sh "git submodule update --init"
    }
  }
}

def per_exec_ws(folder) {
  return "workspace/exec_${env.EXECUTOR_NUMBER}/" + folder
}


def make_tvm(docker_type, path, make_flag) {
  timeout(time: max_time, unit: 'MINUTES') {
    try {
      sh "${docker_run} ${docker_type} ./tests/scripts/task_tvm_build.sh ${path} ${make_flag}"
      // always run cpp test when build
      sh "${docker_run} ${docker_type} ./tests/scripts/task_tvm_cpptest.sh"
    } catch (exc) {
      echo 'Incremental compilation failed. Fall back to build from scratch'
      sh "${docker_run} ${docker_type} ./tests/scripts/task_tvm_clean.sh ${path}"
      sh "${docker_run} ${docker_type} ./tests/scripts/task_tvm_build.sh ${path} ${make_flag}"
      sh "${docker_run} ${docker_type} ./tests/scripts/task_tvm_cpptest.sh"
    }
  }
}


stage("Sanity Check") {
  timeout(time: max_time, unit: 'MINUTES') {
    node('CPU') {
      ws(per_exec_ws("tvm-vta-hw/sanity")) {
        init_git()
        sh "${docker_run} ${ci_lint} ./tests/scripts/task_tvm_checkout.sh"
        sh "${docker_run} ${ci_lint} ./tests/scripts/task_lint.sh"
      }
    }
  }
}

stage("Build") {
  timeout(time: max_time, unit: "MINUTES") {
    node("CPU") {
      ws(per_exec_ws("tvm-vta-hw/build")) {
        init_git()
        sh "${docker_run} ${ci_lint} ./tests/scripts/task_tvm_checkout.sh"
        sh "${docker_run} ${ci_cpu} ./tests/scripts/task_tvm_config_build_cpu.sh"
        make_tvm(ci_cpu, "tvm/build", "-j2")
        // run sim tests
        sh "${docker_run} ${ci_cpu} ./tests/scripts/task_python_vta_fsim.sh"
        sh "${docker_run} ${ci_cpu} ./tests/scripts/task_python_vta_tsim.sh"
      }
    }
  }
}
