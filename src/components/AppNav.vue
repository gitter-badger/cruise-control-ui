<!-- Copyright 2017-2019 LinkedIn Corp. Licensed under the BSD 2-Clause License (the "License"). See License in the project root for license information. -->
<template>
  <nav class="navbar navbar-expand-md navbar-dark fixed-top bg-dark">
    <a href="#" target="_self" class="navbar-brand">
        <img height=48 src="/static/cc-logo.png" alt="LinkedIn Kafka Cruise Control Frontend" />
    </a>
    <ul class="navbar-nav">
      <li class="nav-item dropdown" v-for="(groupm, forgroup) in groups" :key="forgroup">
        <a class="nav-link dropdown-toggle" data-toggle='dropdown'>{{ forgroup }}</a>
        <div class="dropdown-menu">
          <router-link class='dropdown-item' :key="c.label" v-for="c in groupm" :to='{"name": "page.kafkaclusterstate", "params": { "group": forgroup, "cluster": c.label } }'>{{ c.label }}</router-link>
        </div>
      </li>
    </ul>
    <ul class="navbar-nav ml-auto">
      <li right="" class="nav-item">
        <router-link :to='{"name": "preferences"}' class="nav-link" target="_self">&#9881;  Preferences</router-link>
      </li>
    </ul>
  </nav>
</template>

<script>
import xssFilters from 'xss-filters'

export default {
  name: 'AppNav',
  props: {
    group: String,
    cluster: String
  },
  data () {
    return {
      groups: {},
      active: {
        group: null,
        cluster: null,
        url: null
      }
    }
  },
  created () {
    let vm = this
    let notselected = true
    // download the cluster information csv file
    vm.$http.get(vm.$store.state.configurl, {withCredentials: true}).then((r) => {
      let lines = r.data.split(/\n/)
      let groups = {}
      let config = {}
      for (let i = 0; i < lines.length; i++) {
        let csv = lines[i].split(/,/)
        if (csv.length === 3) {
          let group = xssFilters.inHTMLData(csv[0])
          let label = xssFilters.inHTMLData(csv[1])
          let url = xssFilters.uriInHTMLData(csv[2])
          if (!groups[group]) {
            groups[group] = []
          }
          if (!config[group]) {
            config[group] = {}
          }
          if (!config[group][label]) {
            config[group][label] = csv[2]
          }
          groups[group].push({label: label, url: url})
          // set the initial selected things
          if (notselected) {
            vm.$set(vm.active, 'group', group)
            vm.$set(vm.active, 'cluster', label)
            vm.$set(vm.active, 'url', url)
          }
        }
      }
      vm.groups = groups
      vm.$store.commit('config', config)
      if (Object.keys(groups).length === 0) {
        vm.$store.commit('configError', true)
        vm.$store.commit('configErrorMessage', 'No Cruise-Control REST End point information found found in the : ' + vm.$store.state.configurl)
      } else {
        vm.$store.commit('configError', false)
        vm.$store.commit('configErrorMessage', null)
      }
    }, (e) => {
      vm.$store.commit('configError', true)
      vm.$store.commit('configErrorMessage', 'Error encountered while fetching :' + vm.$store.state.configurl)
    })
  }
}
</script>
