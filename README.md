# Example 1 of using the galaxy-k8s-action with ABM 

This is an example repository showing how to use the [galaxy-k8s-action](https://github.com/ksuderman/galaxy-k8s-action), used for installing Galaxy onto Kubernetes, with [abm](https://github.com/galaxyproject/gxabm) used to run Galaxy workflows.

This example defines all of the `abm` configuration in the workflow.  See [this repository](https://github.com/ksuderman/galaxy-k8s-action-example-2) for an exmple of using `galaxy-k8s-action` with `abm` using configuration files included in the repository.

**NOTE** This example requires a larger job runner (available with GitHub Enterrpise) or a self-hosted job runner with at least eight cores.  The standard GitHub job runner with four cores is enough to install Kubernetes and Galaxy, but does not have enough resources to run the benchmark. 

```bash
# Example workflow
name: Run a Galaxy benchmark with ABM
on:
  workflow_dispatch:
    inputs:
      values-file:
        description: 'Values file to use for the Galaxy Helm chart'
        required: true
      workflow:
        description: 'URL to a Galaxy workflow to use for the benchmark'
        default: https://benchmarking-inputs.s3.amazonaws.com/Variant/Galaxy-Workflow-Variant_analysis_on_WGS_PE_data.ga
        required: true
      history:
        description: 'URL to an exported Galaxy history to use for the benchmark'
        default: https://benchmarking-inputs.s3.amazonaws.com/Variant/Variant-calling-inputs---2GB.rocrate.zip
        required: true
jobs:
  install-galaxy:
    name: Install Galaxy and run a benchmark with ABM
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          persist-credentials: false

      - uses: ksuderman/github-action-galaxy-k8s@dev
        with:
          api-key: secret
          values-file: ${{ inputs.values-file }}
          admin-users: admin@example.com

      - name: Get Helm values
        shell: bash
        run: |
          helm get values galaxy -n galaxy --all --output yaml

      - name: Install ABM
        shell: bash
        run: |
          # Install ABM
          pip install gxabm
          abm --version

      - name: Configure ABM and Galaxy
        shell: bash
        run: |        
          # Configure ABM
          mkdir -p .abm
          cat > .abm/profile.yml << EOF
          localhost:
            url: http://localhost
            key: secret
            kube: .kube/config
          EOF
          abm config list
          # Create a Galaxy user and an API key
          key=$(abm localhost user create admin admin@example.com galaxypassword | jq -r .key)
          abm config key localhost $key
          # Import a workflow and a history with appropriate d
          abm localhost workflow import ${{ inputs.workflow }}
          abm localhost history import ${{ inputs.history }}

      - name: Run the benchmark
        timeout-minutes: 30
        shell: bash
        run: |
          cat > benchmark.yml << EOF
            - output_history_base_name: Variant-Calling
              workflow_id: Variant analysis on WGS PE data
              runs:
              - history_name: 2GB
                inputs:
                - name: Paired Collection
                  collection: Subsample of reads from SRR24043307
                - name: GenBank genome
                  dataset_id: GRCh38.p14.gbff.gz
                - name: Name for genome database
                  value: h38          
          EOF
          cat benchmark.yml
          # Validate and run the benchmark
          abm localhost benchmark validate benchmark.yml          
          #abm localhost benchmark run benchmark.yml 

      - name: List the results
        shell: bash
        run: |
          abm localhost history list
          abm localhost jobs list


```