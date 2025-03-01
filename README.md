# Example 1 of using the galaxy-k8s-action with ABM 

This is an example repository showing how to use the [galaxy-k8s-action](https://github.com/ksuderman/galaxy-k8s-action), used for installing Galaxy onto Kubernetes, with [abm](https://github.com/galaxyproject/gxabm) used to run Galaxy workflows.

This example defines all of the `abm` configuration in the workflow.  See [this repository](https://github.com/ksuderman/galaxy-k8s-action-example-2) for an exmple of using `galaxy-k8s-action` with `abm` using configuration files included in the repository.

## Example workflow
```yaml
name: Test a Galaxy benchmark with ABM
on:
  workflow_dispatch:
    inputs:
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
          admin-users: admin@example.com

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
          mkdir -p $HOME/.abm
          abm config create localhost $HOME/.kube/config
          abm config key localhost secret
          abm config url localhost http://localhost
          abm config list
          ls -al .abm
          cat .abm/profile.yml
          # Create a Galaxy user and an API key
          key=$(abm localhost user create admin admin@example.com galaxypassword | jq -r .key)
          abm config key localhost $key
          # Import a workflow, a history, and generate a benchmark
          abm localhost workflow import ${{ inputs.workflow }}
          abm localhost history import ${{ inputs.history }}

      - name: Run the benchmark
        shell: bash
        run: |
          cat > benchmark.yml << EOF
            - output_history_base_name: Variant-Calling
              workflow_id: Variant analysis on WGS PE data
              runs:
              - history_name: 2GB
                inputs:
                - name: Paired Collection
                  collection: SRR24043307-2GB
                - name: GenBank genome
                  dataset_id: GRCh38.p14.gbff.gz
                - name: Name for genome database
                  value: h38          
          EOF
          cat benchmark.yml
          # Validate and run the benchmark
          abm localhost benchmark validate benchmark.yml          
          abm localhost benchmark run benchmark.yml 

      - name: List the results
        shell: bash
        run: |
          abm localhost history list
          abm localhost jobs list

```