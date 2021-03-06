<!-- Copyright 2017-2019 LinkedIn Corp. Licensed under the BSD 2-Clause License (the "License"). See License in the project root for license information. -->
<template>
  <div>
    <div class="alert alert-info" v-if='!hideHelperURL'>
      <b>URL ({{group}}, {{cluster}}):</b> <a target=_blank :href='url'>{{ url }}</a>
    </div>
    <div class="alert alert-primary">
      <form>
        <div class="form-row">
          <div class="form-group col-md-6">
            <label>Resource:</label>
            <select class="form-control" v-model='resource'>
              <option :key='r.key' :value='r.key' v-for='r in allResources'>{{ r.label }}</option>
            </select>
          </div>
          <div class="form-group col-md-6">
            <label for="inputPassword4"># Entries</label>
            <input type="number" class="form-control" v-model='numRecords'>
          </div>
        </div>
        <div class="form-row">
          <div class="form-group col-md-8">
            <label for="inputAddress">Topic <b title="Topics takes  Regular Expression">❓</b></label>
            <input type="text" class="form-control" v-model='topicName'>
          </div>
          <div class="form-group col-md-2">
            <label for="inputAddress">Min Partition #</label>
            <input type="number" min="0" class="form-control" v-model='partitionRangeMin'>
          </div>
          <div class="form-group col-md-2">
            <label for="inputAddress">Max Partition #</label>
            <input type="number" min="0" class="form-control" v-model='partitionRangeMax'>
          </div>
        </div>
        <!-- TODO: Use date/time picker which requies jquery
        <div class="form-row">
          <div class="form-group col-md-6">
            <label for="inputAddress">Start Time</label>
            <input type="number" min=0 class="form-control" v-model='startTimestamp'>
          </div>
          <div class="form-group col-md-6">
            <label for="inputAddress">End Time</label>
            <input type="number" min=0 class="form-control" v-model='endTimestamp'>
          </div>
        </div>
        -->
        <div class="form-group">
          <div class="form-check">
            <input class="form-check-input" type="checkbox" v-model='allowCapacityEstimation'>
            <label class="form-check-label">
              Allow Capacity Estimation
            </label>
          </div>
        </div>
        <div class="form-group">
          <div class="form-check">
            <input class="form-check-input" type="checkbox" v-model='maxLoad'>
            <label class="form-check-label">
              Max Load
            </label>
          </div>
        </div>
        <div class="form-group">
          <div class="form-check">
            <input class="form-check-input" type="checkbox" v-model='showURL'>
            <label class="form-check-label">
              Show URL (Debug)
            </label>
          </div>
        </div>

        <button class="btn btn-primary" @click.prevent='acceptTos()'>Show Partition Load</button>
        <code v-if='showURL'>
        <hr>
        {{ url }}
        </code>
      </form>
      <!--
        GET /kafkacruisecontrol/partition_load?
        resource=[RESOURCE]&
        start=[START_TIMESTAMP]&
        end=[END_TIMESTAMP]&
        json=[true/false]&
        entries=[MAX_NUMBER_OF_PARTITION_LOAD_ENTRIES_TO_RETURN]&
        topic=[TOPIC]&
        partition=[START_PARTITION_INDEX-END_PARTITION_INDEX]&
        allow_capacity_estimation=[true/false]&
        min_valid_partition_ratio=[PERCENTAGE]&
        max_load=[true/false]
      -->
    </div>
    <div v-if="error">
      <exception :exception='errorData'></exception>
    </div>
    <div v-else-if='async'>
      <async-task :asyncData='asyncData'></async-task>
    </div>
    <div v-else-if="!loaded && loading">
      Loading ...
    </div>
    <table class="table table-sm table-bordered" v-else-if='records.length > 0'>
      <thead>
        <tr>
          <th :key='h' v-for="h in header">{{ h | camelCase }}</th>
        </tr>
      </thead>
      <transition-group name="fade" tag="tbody">
        <tr :key='r.topic + "-" + r.partition' v-for="r in records">
          <td v-for='(hv, hk) in header'>
            <template v-if='colUnits[hk] === "float"'>
              {{ r[hv].toFixed(2) }}
            </template>
            <template v-else-if='colUnits[hk] === "int"'>
              {{ parseInt(r[hv], 10) }}
            </template>
            <template v-else>
              {{ r[hv] }}
            </template>
          </td>
        </tr>
      </transition-group>
    </table>
    <div class="alert alert-warning" v-if='loaded && records.length == 0'>No Records Found.</div>
  </div>
</template>

<script>
import xssFilters from 'xss-filters'
import Exception from '@/components/Exception'

export default {
  name: 'PartitionLoad',
  props: {
    group: String,
    cluster: String
  },
  components: {
    'exception': Exception
  },
  data () {
    return {
      title: 'Load',
      loaded: false,
      loading: false,
      tos: false,
      error: false,
      errorData: null,
      async: false,
      asyncData: null,
      // Form Elements
      resource: 'DISK',
      allResources: [
        {key: 'CPU', label: 'CPU'},
        {key: 'NW_IN', label: 'Network In'},
        {key: 'NW_OUT', label: 'Network Out'},
        {key: 'DISK', label: 'Disk'}
      ],
      numRecords: 100,
      allowCapacityEstimation: true,
      maxLoad: false,
      topicName: '',
      partitionRangeMin: 0,
      partitionRangeMax: 0,
      startTimestamp: 0,
      endTimestamp: 0,
      minValidPartitionRatio: 0,
      showURL: false,
      // header & records
      records: [],
      header: [],
      colUnits: []
    }
  },
  watch: {
    group: function (ogroup, ngroup) {
      this.argsChanged()
    },
    cluster: function (ocluster, ncluster) {
      this.argsChanged()
    }
  },
  computed: {
    apiMinorVersion () {
      // NnwOutRate has been changed to NwOutRate and Upstream
      // API does not expose this correctly.
      if (this.records.length > 0 && this.records[0].hasOwnProperty('networkInbound')) {
        return 2
      } else {
        return 1
      }
    },
    hideHelperURL () {
      return this.$store.state.hideHelperURL
    },
    url () {
      let params = {
        resource: this.resource,
        entries: this.numRecords,
        allow_capacity_estimation: this.allowCapacityEstimation,
        max_load: this.maxLoad
      }
      if (this.topicName.length > 1) {
        params.topic = this.topicName.replace(/^\s+|\s+$/, '')
        params.topic = xssFilters.uriQueryInHTMLData(params.topic)
      }
      if (this.partitionRangeMin > 0 && this.partitionRangeMax > this.partitionRangeMin) {
        params.partition = this.partitionRangeMin + '-' + this.partitionRangeMax
      }
      if (this.startTimestamp > 0 && this.endTimestamp > this.startTimestamp) {
        params.start = this.startTimestamp
        params.end = this.endTimestamp
      }
      if (this.minValidPartitionRatio > 0) {
        params.min_valid_partition_ratio = this.minValidPartitionRatio
      }
      return this.$helpers.getURL('partitionload', params)
    }
  },
  methods: {
    acceptTos () {
      this.tos = true
      this.getPartitionLoad()
    },
    argsChanged () {
      const newurl = this.$store.getters.getnewurl(this.group, this.cluster)
      this.$store.commit('seturl', newurl)
      this.loaded = false
      this.newurl = newurl
      this.getPartitionLoad()
    },
    getPartitionLoad () {
      let vm = this
      vm.loaded = false
      vm.loading = true
      vm.$http.get(vm.url, {withCredentials: true}).then((r) => {
        if (r.headers['content-type'].match(/text\/plain/) || r.data.progress) {
          vm.async = true
          vm.asyncData = r.data
        } else {
          vm.async = false
          vm.loading = false
          vm.loaded = true
          vm.error = false
          vm.errorData = null
          vm.records = r.data.records || []
          // CPU/Disk/NW* are converted to lowercase after refactor in CruiseControl code
          if (vm.apiMinorVersion === 2) {
            vm.header = ['topic', 'partition', 'leader', 'followers', 'cpu', 'disk', 'networkInbound', 'networkOutbound', 'msg_in']
            vm.colUnits = ['str', 'int', 'int', 'list', 'float', 'float', 'float', 'float', 'int']
          } else {
            vm.header = ['topic', 'partition', 'leader', 'followers', 'CPU', 'DISK', 'NW_IN', 'NW_OUT', 'MSG_IN']
            vm.colUnits = ['str', 'int', 'int', 'list', 'float', 'float', 'float', 'float', 'int']
          }
        }
      }, (e) => {
        vm.error = true
        vm.loading = false
        vm.errorData = e && e.response ? e.response.data : e
      })
    }
  }
}
</script>

<style scoped>
.fade-enter-active, .fade-leave-active {
  transition: opacity .5s
}
.fade-enter, .fade-leave-to  {
  opacity: 0
}
</style>
