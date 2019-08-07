<template>
  <div class='demo_container'>
    <button v-on:click="plot_click">Plot Data</button>
    <line-chart-comp-d3
        title_1="Votes for Republican Candidate in Presidential Elections"
        title_2="(Kentucky vs New York)"
        :chart_data="chart_data"
        :axis="axis"
        :points="points"
        :margin_bottom="margin_bottom"
        :css_variables="css_variables">
      <svg class="svg_chart_1" width="1200" height="700"></svg>
    </line-chart-comp-d3>
  </div>
</template>

<script>
  import LineChartCompD3 from 'linechartcompd3';
  import {read_csv} from 'd3fetchmodule';

  export default {
    name: 'App',
    data: function() {
      return {
        chart_data: null,
        axis: {
          x: {
            title: 'Year',
            data_type: 'time',
            key: 'year',
            tic_format: '%Y',
            tic_rotation: 0,
            value_format: '%Y',
          },
          y: {
            title: 'Percent',
            data_type: 'linear',
            key: 'percent',
            tic_format: '.0f',
            value_format: '.1f',
          }
        },
        points: [
          {
            text: 'Kentucky',
            select:
              function(d) {
                if(d['state'] === 'Kentucky') {
                  return true;
                }
              },
            icon: '\u2605',
            color: 'red',
            connect: true
          },
          {
            text: 'New_York',
            select:
              function(d){
                if(d['state'] === 'New_York') {
                  return true;
                }
              },
            //      icon: '\u2206',
            color: 'yellow',
            connect: true
          }
        ],
        margin_bottom: 55,
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

          select_comp_color: 'white',
          select_comp_border_color: 'white',
          select_comp_heading_color: 'white',
          select_comp_arrow_color: 'white',
          select_comp_items_panel_color: 'white',
          select_comp_items_panel_background: 'black',
          select_comp_items_panel_border: 'white',

          button_comp_font_size: '12px'
        }
      };
    },
    components: {
      LineChartCompD3
    },
    methods: {
      plot_click: function(){
        const get_data = async () => {
          try {
            const convert = [
              {field: 'year', type: 'time', time_format: '%Y'},
              {field: 'percent', type: 'linear'},
              {field: 'state', type: 'band'}
            ];
            this.chart_data = await read_csv('data/votes.csv',convert);
            //debug
            console.log(JSON.stringify(this.chart_data));
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
</style>