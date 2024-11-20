<template>
  <div>
    <!-- Loading -->
    <div class="text-center q-pa-xl" v-show="showLoader">
      <div>
        <q-spinner-oval size="lg" />
      </div>
      <div class="text-caption">
        Carregando...
      </div>
    </div>

    <!-- Empty -->
    <div class="text-center q-pa-xl" v-show="!showLoader && RawData.length == 0">
      <div>
        <q-icon size="lg" name="far fa-folder-open"></q-icon> *
      </div>
      <div class="text-h6">
        Sem Dados.
      </div>
    </div>

    <!-- Content -->
    <div v-show="!showLoader && RawData.length > 0" :id="`chart-${Name}-container`">
      <canvas :style="this.CanvasStyle" :id="`chart-${Name}-canvas`"></canvas>
    </div>
  </div>
</template>

<script>
// Services:
import Utils from '../../../services/utils'

// Libs:
import Chart from 'chart.js'

export default {
  name: 'components-common-charts-pie',

  props: {
    Name: String,
    RawData: Object,
    LoadDataFn: Function,
    Filter: Function,
    Datasets: Object,
    LabelsField: String,
    HideLegend: Boolean,
    CanvasStyle: String,
    Percentage: Boolean,
  },

  data() {
    return {
      chartElement: null,
      loading: false,
      showLoader: false
    }
  },

  watch: {
    loading: {
      handler: function (v) {
        this.showLoader = v;
      }
    },

    RawData: {
      handler: function () {
        this.triggerChart();
      },
      deep: true
    },
  },

  methods: {
    async loadData() {
      if (!this.loading) {
        this.loading = true;

        var params = {};

        if (!!this.Filter) params = this.Filter();

        await this.LoadDataFn(params);
      }
    },

    datasets() {
      var datasets = [];
      for (let i = 0; i < this.Datasets.length; i++) {
        let set = this.Datasets[i];

        datasets.push({
          label: set.label,
          data: [],
          backgroundColor: [],
          borderColor: [],
          borderWidth: 1
        })
      }

      return datasets;
    },

    triggerChart() {
      var chartObj = {
        type: "pie",
        data: {
          labels: [],
          datasets: this.datasets()
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          legend: { display: !this.HideLegend },
          tooltips: {
            mode: 'label',
            callbacks: {
              label: (tooltipItem, data) => {
                var index = tooltipItem.datasetIndex;
                var value = data.datasets[index].data[tooltipItem.index];

                if (!!this.Datasets[index].formatTooltip && typeof this.Datasets[index].formatTooltip == 'function')
                  return this.Datasets[index].formatTooltip(data.datasets[index].label, value);
                else return `${data.datasets[index].label}: ${value}${this.Percentage ? '%' : ''}`;
              }
            }
          }
        },

      };

      // Clone datasets, detaching from reference:
      var datasets = JSON.parse(JSON.stringify(chartObj.data.datasets));
      var labels = [];
      var bgColors = [];
      var strokeColors = [];

      if (this.RawData[0] instanceof Array) {
        let setLabels = true;
        for (let j = 0; j < this.Datasets.length; j++) {
          let datasetData = [];
          for (let i = 0; i < this.RawData[j].length; i++) {
            let data = this.RawData[j][i];
            let value = data[this.Datasets[j].field];

            if (setLabels)
              labels.push(data[this.LabelsField]);

            let color = Utils.randomHexColor();
            bgColors.push(color);
            strokeColors.push(color);

            datasetData.push(value);
          }
          if (this.Percentage) {
            let whole = datasetData.sumValues();

            for (let idx = 0; idx < datasetData.length; idx++) {
              datasetData[idx] = ((datasetData[idx] / whole) * 100).toFixed(2);
            }
          }

          setLabels = false;
          datasets[j].data = datasetData;
          datasets[j].backgroundColor = bgColors;
          datasets[j].borderColor = strokeColors;
        }
      } else {
        for (let i = 0; i < this.RawData.length; i++) {
          let data = this.RawData[i];
          labels.push(data[this.LabelsField]);

          for (let j = 0; j < this.Datasets.length; j++) {
            let dataset = this.Datasets[j];
            let indicator = !!dataset.indicator ? dataset.indicator.split('=') : null;

            if (!!indicator && data[indicator[0]] != indicator[1])
              continue;

            datasets[j].data.push(data[dataset.field]);
          }
        }
      }

      chartObj.data.labels = labels;
      // Update reference with (re)created datasets:
      chartObj.data.datasets = datasets;

      // Update chart:
      if (this.chartElement != null) {
        this.chartElement.config = chartObj;
        this.chartElement.update();
      }
      // Start new chart: 
      else {
        var ctx = document.getElementById(`chart-${this.Name}-canvas`);
        this.chartElement = new Chart(ctx, chartObj);
      }

      this.loading = false;
    },
  },

  mounted() {
    this.loadData();

    this.$emit('reload-fn', this.loadData);
  }
}
</script>