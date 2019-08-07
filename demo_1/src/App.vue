<template>
  <div class='demo_container'>
    <section class="button_sec">
      <button v-on:click="plot_1_click">Plot petal_length vs sepal_length</button>
      <button v-on:click="plot_2_click">Plot petal_width vs sepal_width</button>
    </section>
    <line-chart-comp-d3
        :title_1=title_1
        :title_2=title_2
        :chart_data="chart_data"
        :axis="axis"
        :points="points"
        :fits="fits"
        :css_variables="css_variables">
      <svg class="svg_chart_1" width="1200" height="700"></svg>
    </line-chart-comp-d3>
  </div>
</template>

<script>
  import LineChartCompD3 from 'linechartcompd3';
  import {read_csv} from 'd3fetchmodule';
  import regression from 'regression';

  export default {
    name: 'App',
    data: function() {
      return {
        title_1: null,
        title_2: null,
        chart_data: null,
        fits: null,
        axis: null,
        points: null,
        css_variables: {
          line_chart_compD3_color: 'white',
          line_chart_compD3_background_color: 'black',
          line_chart_compD3_axis_font_size: '12px',
          line_chart_compD3_axis_color: 'white',
          line_chart_compD3_point_opacity: '0.7',
          line_chart_compD3_tooltip_fill: 'white',

          input_comp_heading_color: 'white',
          input_comp_heading_font_size: '12px',
          input_comp_input_color: 'white',
          input_comp_input_border_color: 'white',
          input_comp_input_placeholder_color: 'white',
          input_comp_input_focus_background: 'black',

          button_comp_font_size: '12px'
        }
      };
    },
    components: {
      LineChartCompD3
    },
    methods: {
      plot_1_click: function(){
        this.title_1 = 'Fisher Iris Flower Data';
        this.title_2 = '(petal_length vs sepal_length)';
        this.axis = {
          x: {
            title: 'Petal Length',
            data_type: 'linear',
            key: 'petal_length',
            tic_format: '.1f',
            tic_rotation: 0,
            value_format: '.1f'
          },
          y: {
            title: 'Sepal Length',
            data_type: 'linear',
            key: 'sepal_length',
            tic_format: '.1f',
            value_format: '.1f'
          }
        };
        this.points = [
          {
            text: 'Iris-setosa',
            select:
              function(d) {
                if(d['class'] === 'Iris-setosa') {
                  return true;
                }
              },
            icon: '\u2605',
            color: 'red',
            connect: false
          },
          {
            text: 'Iris-versicolor',
            select:
              function(d){
                if(d['class'] === 'Iris-versicolor') {
                  return true;
                }
              },
            icon: '\u26AB',
            color: 'white',
            connect: false
          },
          {
            text: 'Iris-virginica',
            select:
              function(d){
                if(d['class'] === 'Iris-virginica') {
                  return true;
                }
              },
            icon: '\u2206',
            color: 'yellow',
            connect: false
          }
        ];
        const get_data = async () => {
          const convert = [
            {field: 'petal_length', type: 'linear'},
            {field: 'sepal_length', type: 'linear'},
            {field: 'class', type: 'head'}
          ];
          try {
            this.chart_data = await read_csv('data/iris.csv',convert);
            //debug
            console.log(JSON.stringify(this.chart_data));
            //define 2 fits
            const regress_data = [];
            this.fits = [];
            for(let row of this.chart_data){
              regress_data.push([row['petal_length'],row['sepal_length']]);
            }
            const regress_1 = regression.linear(regress_data);
            const fit_1 = {};
            fit_1.text = regress_1.string + `  (R2: ${regress_1.r2})`;
            fit_1.equation = regress_1.predict;
            fit_1.number_of_points = 50;
            fit_1.color = 'violet';
            this.fits.push(fit_1);

            const regress_2 = regression.polynomial(regress_data);
            const fit_2 = {};
            fit_2.text = regress_2.string + `  (R2: ${regress_2.r2})`;
            fit_2.equation = regress_2.predict;
            fit_2.number_of_points = 100;
            fit_2.color = 'lime';
            this.fits.push(fit_2);
          }catch(e){
            console.log(e);
          }
        };
        get_data();
      },
      plot_2_click: function(){
        this.fits = null;
        this.title_1 = 'Fisher Iris Flower Data';
        this.title_2 = '(petal_width vs sepal_width)';
        const convert = [
          {field: 'petal_width', type: 'linear'},
          {field: 'sepal_width', type: 'linear'},
          {field: 'class', type: 'head'}
        ];
        this.axis = {
          x: {
            title: 'Petal Width',
            data_type: 'linear',
            key: 'petal_width',
            tic_format: '.1f',
            tic_rotation: 0,
            value_format: '.1f'
          },
          y: {
            title: 'Sepal Width',
            data_type: 'linear',
            key: 'sepal_width',
            tic_format: '.1f',
            value_format: '.1f'
          }
        };
        this.points = [
          {
            text: 'Iris-versicolor',
            select:
              function(d){
                if(d['class'] === 'Iris-versicolor'){
                  return true;
                }
              },
            icon: '\u26AB',
            color: 'white',
            connect: false
          },
          {
            text: 'Iris-virginica',
            select:
              function(d){
                if(d['class'] === 'Iris-virginica'){
                  return true;
                }
              },
            icon: '\u2206',
            color: 'yellow',
            connect: false
          }
        ];
        const get_data = async () => {
          try{
            this.chart_data = await read_csv('data/iris.csv', convert);
            //debug
            console.log(JSON.stringify(this.chart_data));
            //define 1 linear fit
            const regress_data = [];
            this.fits = [];
            for(let row of this.chart_data){
              if(row['class'] === 'Iris-versicolor' || row['class'] === 'Iris-virginica'){
                regress_data.push([row['petal_width'], row['sepal_width']]);
              }
            }
            const regress_1 = regression.linear(regress_data);
            const fit_1 = {};
            fit_1.text = regress_1.string + `  (R2: ${regress_1.r2})`;
            fit_1.equation = regress_1.predict;
            fit_1.number_of_points = 25;
            fit_1.color = 'violet';
            this.fits.push(fit_1);
          }catch(e){
            console.log(e);
          }
        };
        get_data();
      }
    }
  }
</script>

<style>
  .demo_container {
    background-color: black;
  }
  .button_sec {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    width: 300px;
  }
</style>