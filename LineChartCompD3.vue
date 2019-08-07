<template>
  <div>
    <div class="lineChartCompD3">
      <section :class="[scaling_is_active ? 'lineChartCompD3_scaling' : 'lineChartCompD3_not_scaling']">
        <input-comp
            header_position="above"
            placeholder="Enter min"
            input_size="7"
            :heading="head_min"
            :input_value="min"
            :single_border="single_border"
            v-on:input_comp_value_changed="value => {this.min = value}">
        </input-comp>
        <input-comp
            header_position="above"
            placeholder="Enter max"
            input_size="7"
            :heading="head_max"
            :input_value="max"
            :single_border="single_border"
            v-on:input_comp_value_changed="value => {this.max = value}">
        </input-comp>
        <input-comp
            header_position="above"
            placeholder="Enter step"
            input_size="7"
            :heading="head_step"
            :input_value="step"
            :single_border="single_border"
            v-on:input_comp_value_changed="value => {this.step = value}">
        </input-comp>
        <select-comp
            v-show="doing_time"
            heading="step time unit"
            placeholder="Select step time"
            :select_value="select_step_time"
            :items="time_intervals"
            :single_border="single_border"
            :blur_panel="blur_panel"
            :css_variables="select_css_variables"
            v-on:select_comp_value_changed="value => {this.select_step_time = value}">
        </select-comp>
        <input-comp
            header_position="above"
            placeholder="Enter tick format"
            input_size="14"
            :heading="head_tick_format"
            :input_value="tick_format"
            :single_border="single_border"
            v-on:input_comp_value_changed="value => {this.tick_format = value}">
        </input-comp>
        <button-comp
            :css_variables="button_css_variables"
            v-on:button_comp_clicked="update_clicked">Update
        </button-comp>
        <button-comp
            :css_variables="button_css_variables"
            v-on:button_comp_clicked="reset_clicked">Reset
        </button-comp>
      </section>
      <section class="lineChartCompD3_svg__sec">
        <slot></slot>
      </section>
    </div>
  </div>
</template>

<script>
  import ButtonComp from 'buttoncomp';
  import InputComp from 'inputcomp';
  import SelectComp from 'selectcomp';
  import {select,selectAll,local} from 'd3-selection';
  import {line} from 'd3-shape';
  import {max,range} from 'd3-array';
  import {format} from 'd3-format';
  import {timeParse,timeFormat} from 'd3-time-format';
  import {define_axis,update_axis} from 'd3axismodule';
  import {define_scale,minmax,calculate_scale,time_interval} from 'd3ScaleModule';
  import {make_draggable,init_chart} from 'd3utilmodule';

  export default {
    name: 'LineChartCompD3',
    data(){
      return {
        svg: null,
        grouped_data: null,
        filtered_data: null,
        min: null,
        max: null,
        step: null,
        tick_format: null,
        current_min: {x: null, y: null},
        current_max: {x: null, y: null},
        current_step: {x: null, y: null},
        current_tick_format: {x: null, y: null},
        current_axis: null,
        initials: {x: {min: null, max: null, tick_values: null},
          y: {min: null, max: null, tick_values: null}},
        head_min: null,
        head_max: null,
        head_step: null,
        head_tick_format: null,
        chart_axis: {x: null,y: null},
        scale: {x: null,y: null},
        pixels: {x: null,y: null},
        chart_g: null,
        doing_time: false,
        time_intervals: ['auto','year','month','week','day','hour','minute','second','millisecond'],
        scaling_is_active: false,
        connected_points: null,
        icons: null,
        tooltips_text: null,
        alocal: null,

        button_css_variables: {
          button_comp_font_size: '.6rem'
        },

        //input-comp stuff
        single_border: true,

        //select=comp stuff
        select_step_time: 'auto',
        blur_panel: false,
        select_css_variables: {
          select_comp_items_panel_position: 'absolute',
          select_comp_items_panel_z_index: '100'
        },

        //define line generators
        line_generator: line()
          .x(d => {
            return this.scale.x(d[0]);
          })
          .y(d => {
            return this.scale.y(d[1]);
          }),
        line_generator_obj: line()
          .x(d => {
            return this.scale.x(d[this.axis.x.key]);
          })
          .y(d => {
            return this.scale.y(d[this.axis.y.key]);
          })
      }
    },
    props: {
      chart_data: {
        type: Array,
        default: null
      },
      axis: {
        type: Object,
        default: null
      },
      points: {
        type: Array,
        default: null
      },
      fits: {
        type: Array,
        default: null
      },
      title_1: {
        type: String,
        default: null
      },
      title_2: {
        type: String,
        default: null
      },
      margin_left: {
        type: Number,
        default: 80
      },
      margin_bottom: {
        type: Number,
        default: 50
      },
      css_variables: {
        type: Object,
        default: null
      }
    },
    watch: {
      chart_data: function(new_data){
        if(new_data !== null){
          //create groups of data
          this.grouped_data = [];
          this.filtered_data = [];
          for(let point of this.points){
            const group = new_data.filter(point.select);
            this.filtered_data = this.filtered_data.concat(group);
            this.grouped_data.push(group);
          }

          //remove all current elements
          this.remove_elements();

          //define x scaling and axis
          this.define_x_scale_axis();

          //define y scaling and axis
          this.define_y_scale_axis();

          //draw points if icon from this.points are defined
          this.draw_points();

          //draw axis titles
          this.draw_axis_titles();

          //draw main titles
          this.draw_main_titles();

          //draw connect lines
          this.draw_connect_lines();

          //draw legend
          this.draw_legend();

          //draw fitted lines
          this.draw_fit_lines();

          //create text toottip
          this.create_tooltips();
        }
      }
    },
    methods: {
      define_x_scale_axis: function(){
        let x_nice_scale = null;
        let x_nice_min_max = null;
        let tick_values = null;

        //compute overall raw x min/max of numeric data
        switch(this.axis.x.data_type){
          case 'linear':{
            const x_min_max = minmax(this.filtered_data, [this.axis.x.key]);
            x_nice_scale = calculate_scale(x_min_max[0],x_min_max[1]);
            x_nice_min_max = [x_nice_scale.min,x_nice_scale.max];
            //update currents
            this.set_currents('x',
              x_nice_scale.min,
              x_nice_scale.max,
              x_nice_scale.step,
              this.axis.x.tic_format
            );
            break;
          }
          case 'time': {
            x_nice_min_max = minmax(this.filtered_data, [this.axis.x.key]);
            //update currents
            this.set_currents('x',
              x_nice_min_max[0],
              x_nice_min_max[1],
              'auto',
              this.axis.x.tic_format
            );
            break;
          }
          case 'band':
            x_nice_min_max = this.axis.x.bands;
            break;
        }

        //define x scaling
        this.scale.x = define_scale(
          this.axis.x.data_type,
          'x',
          x_nice_min_max,
          this.pixels.x
        );

        //define tick_values
        if(this.axis.x.data_type === 'time'){
          tick_values = this.scale.x.ticks();
        }else if(this.axis.x.data_type === 'band'){
          tick_values = this.axis.x.bands;
        }else{
          tick_values = range(
            this.scale.x.domain()[0],
            this.scale.x.domain()[1],
            x_nice_scale.step
          );
        }

        //define initials
        this.set_initials('x',
          x_nice_min_max[0],
          x_nice_min_max[1],
          tick_values,
          this.axis.x.tic_format
        );

        //define x axis
        this.chart_axis.x = define_axis(
          'bottom',
          this.axis.x.data_type,
          'lineChartCompD3_axis lineChartCompD3_axis_x',
          this.scale.x,
          this.chart_g,
          this.pixels.y,
          this.axis.x.tic_format,
          tick_values
        );

        //add click event to x axis
        const axis_group = this.chart_g.select('.lineChartCompD3_axis_x');
        axis_group
          .on('click',() => {
            this.current_axis = 'x';
            this.head_min = 'x minimum';
            this.head_max = 'x maximum';
            this.head_step = 'x step';
            this.head_tick_format = 'x tick format';
            this.min = this.current_min.x;
            this.max = this.current_max.x;
            this.step = this.current_step.x;
            this.tick_format = this.current_tick_format.x;
            if(this.axis.x.data_type === 'time'){
              this.doing_time = true;
            }
            this.scaling_is_active = !this.scaling_is_active;
          });

        //address tic label rotation on x axis
        let x_tic_rotation = 0.0;
        let x_tic_dx = 0.0;
        let x_tic_dy = 0.6;
        let x_tic_anchor = 'middle';
        if(this.axis.x.tic_rotation !== undefined && this.axis.x.tic_rotation !== 0){
          x_tic_rotation = this.axis.x.tic_rotation;
          x_tic_dx = -0.8;
          x_tic_dy = 0.15;
          x_tic_anchor = 'end';
        }
        const x_axis_group = this.chart_g.select('.lineChartCompD3_axis_x');
        x_axis_group
          .selectAll('text')
          .attr('text-anchor', x_tic_anchor)
          .attr('dx', `${x_tic_dx}em`)
          .attr('dy', `${x_tic_dy}em`)
          .attr('transform', `rotate(${x_tic_rotation})`);
      },
      define_y_scale_axis: function(){
        let y_nice_scale = null;
        let y_nice_min_max = null;
        let tick_values = null;

        switch(this.axis.y.data_type){
          case 'linear':{
            const y_min_max = minmax(this.filtered_data, [this.axis.y.key]);
            y_nice_scale = calculate_scale(y_min_max[0],y_min_max[1]);
            y_nice_min_max = [y_nice_scale.min,y_nice_scale.max];
            //update currents
            this.set_currents('y',
              y_nice_scale.min,
              y_nice_scale.max,
              y_nice_scale.step,
              this.axis.y.tic_format
            );
            break;
          }
          case 'band':
            y_nice_min_max = this.axis.y.bands;
            break;
        }

        //define y scaling
        this.scale.y = define_scale(
          this.axis.y.data_type,
          'y',
          y_nice_min_max,
          this.pixels.y
        );

        //define tick_values
        if(this.axis.y.data_type === 'band'){
          tick_values = this.axis.y.bands;
        }else{
          tick_values = range(
            this.scale.y.domain()[0],
            this.scale.y.domain()[1],
            y_nice_scale.step
          );
        }

        //define initials
        this.set_initials('y',
          y_nice_min_max[0],
          y_nice_min_max[1],
          tick_values,
          this.axis.y.tic_format
        );

        //define y axis
        this.chart_axis.y = define_axis(
          'left',
          this.axis.y.data_type,
          'lineChartCompD3_axis lineChartCompD3_axis_y',
          this.scale.y,
          this.chart_g,
          this.pixels.y,
          this.axis.y.tic_format,
          tick_values
        );

        //add click event to y axis
        const axis_group = this.chart_g.select('.lineChartCompD3_axis_y');
        axis_group
          .on('click',() => {
            this.current_axis = 'y';
            this.head_min = 'y minimum';
            this.head_max = 'y maximum';
            this.head_step = 'y step';
            this.head_tick_format = 'y tick format';
            this.min = this.current_min.y;
            this.max = this.current_max.y;
            this.step = this.current_step.y;
            this.tick_format = this.current_tick_format.y;
            this.doing_time = false;
            this.scaling_is_active = !this.scaling_is_active;
          });
      },
      update_clicked: function(){
        //redefine domain scaling for this.current_axis ('x' or 'y')
        let tick_values = null;
        let nice_minmax = null;
        let data_type = this.axis[this.current_axis].data_type;

        const axis_group = this.chart_g.select('.lineChartCompD3_axis_' + this.current_axis);

        switch(data_type){
          case 'linear':{
            const min = parseFloat(this.min);
            const max = parseFloat(this.max);
            tick_values = range(min,max,parseFloat(this.step));
            nice_minmax = [min,max];

            //update scaling
            this.scale[this.current_axis].domain(nice_minmax);

            //update axis
            update_axis(
              this.chart_axis[this.current_axis],
              axis_group,
              this.scale[this.current_axis],
              'linear',
              this.tick_format,
              tick_values
            );

            //update currents
            this.set_currents(
              this.current_axis,
              nice_minmax[0],
              nice_minmax[1],
              parseFloat(this.step),
              this.tick_format);
            break;
          }
          case 'time': {
            const parseTime = timeParse(this.tick_format);
            nice_minmax = [parseTime(this.min),parseTime(this.max)];

            //update scaling
            this.scale[this.current_axis].domain(nice_minmax);

            //update axis
            if(this.select_step_time !== 'auto'){
              //update currents
              this.set_currents(this.current_axis,nice_minmax[0],nice_minmax[1],parseFloat(this.step),this.tick_format);
              const tick_generator = time_interval[this.select_step_time].every(this.current_step[this.current_axis]);
              tick_values = tick_generator.range(
                this.scale[this.current_axis].domain()[0],
                this.scale[this.current_axis].domain()[1]);
            }else {
              //update currents
              this.set_currents(
                this.current_axis,
                nice_minmax[0],nice_minmax[1],
                'auto',
                this.tick_format
              );
              tick_values = this.scale[this.current_axis].ticks();
            }
            update_axis(
              this.chart_axis[this.current_axis],
              axis_group,
              this.scale[this.current_axis],
              'time',
              this.tick_format,
              tick_values
            );
            break;
          }
          case 'band': {
            //update scaling
            //TODO this needs updating:
            this.scale[this.current_axis].domain(this.axis[this.current_axis].bands);
            //update axis
            update_axis(
              this.chart_axis[this.current_axis],
              axis_group,
              this.scale[this.current_axis],
              null,
              null,
              this.axis[this.current_axis].bands
            );
          }
        }

        //redraw the points with updated scaling
        this.redraw_points();
        //redraw fit line with updated scaling
        this.redraw_fit_lines();
        //redraw connect lines
        this.redraw_connect_lines();
      },
      reset_clicked: function(){
        //reset x scale
        this.scale.x.domain([this.initials.x.min,this.initials.x.max]);
        //update axis
        const x_axis_group = this.chart_g.select('.lineChartCompD3_axis_x');

        update_axis(
          this.chart_axis.x,
          x_axis_group,
          this.scale.x,
          this.axis.x.data_type,
          this.initials.x.tick_format,
          this.initials.x.tick_values
        );

        //update x currents
        this.set_currents('x',
          this.initials.x.min,
          this.initials.x.max,
          'auto',
          this.initials.x.tick_format
        );
        if(this.axis.x.data_type !== undefined && this.axis.x.data_type === 'time'){
          this.select_step_time = 'auto';
        }
        //reset y scale
        this.scale.y.domain([this.initials.y.min,this.initials.y.max]);
        //update axis
        const y_axis_group = this.chart_g.select('.lineChartCompD3_axis_y');
        update_axis(
          this.chart_axis.y,
          y_axis_group,
          this.scale.y,
          this.axis.y.data_type,
          this.initials.y.tick_format,
          this.initials.y.tick_values
        );

        //update y currents
        this.set_currents('y',
          this.initials.y.min,
          this.initials.y.max,
          'auto',
          this.initials.y.tick_format
        );

        //redraw the points with updated scaling
        this.redraw_points();
        //redraw fit line with updated scaling
        if(this.fits !== null){
          this.redraw_fit_lines();
        }
        //redraw the connects
        this.redraw_connect_lines();
      },
      draw_points: function(){
        //draw the points(i.e. icons), do we have icons?
        const self = this;
        this.icons = [];
        for(let point of this.points){
          if(Boolean(point.icon)){
            this.icons.push(point);
          }
        }
        if(this.icons.length > 0){
          const points = this.points;
          this.alocal = new local();
          const all_points_g = this.chart_g.append('g')
            .attr('class', 'lineChartCompD3_all_points_g');
          all_points_g
            .selectAll('g')
            .data(this.grouped_data).enter()
            .append('g')
            .attr('class','lineChartCompD3_points_g')
            .selectAll('text')
            .data(function(d, i, group){
              self.alocal.set(this, i);
              return d;
            }).enter()
            .append('text')
            .attr('class', 'lineChartCompD3_points_text')
            .attr('x', d => {
              return this.scale.x(d[this.axis.x.key]);
            })
            .attr('y', d => {
              return this.scale.y(d[this.axis.y.key]);
            })
            .attr('dy', '.35em')
            .attr('fill', function(){
              return points[self.alocal.get(this)].color;
            })
            .text(function(){
              return points[self.alocal.get(this)].icon;
            })
            .on('mouseover', (d) => {
              let tooltip_text_x = null;
              let tooltip_text_y = null;
              //x text value
              if(this.axis.x.data_type === 'time'){
                tooltip_text_x = ` ${this.axis.x.key}:${timeFormat(this.axis.x.value_format)(d[this.axis.x.key])}`;
              }else{
                tooltip_text_x = ` ${this.axis.x.key}:${format(this.axis.x.value_format)(d[this.axis.x.key])}`;
              }
              //y text value
              if(this.axis.y.data_type === 'time'){
                tooltip_text_y = ` ${this.axis.y.key}:${timeFormat(this.axis.y.value_format)(d[this.axis.y.key])}`;
              }else{
                tooltip_text_y = ` ${this.axis.y.key}:${format(this.axis.y.value_format)(d[this.axis.y.key])}`;
              }
              this.tooltips_text.x
                .attr('x', this.scale.x(d[this.axis.x.key]) + 8)
                .attr('y', this.scale.y(d[this.axis.y.key]))
                .text(tooltip_text_x);
              this.tooltips_text.y
                .attr('x', this.scale.x(d[this.axis.x.key]) + 8)
                .attr('y', this.scale.y(d[this.axis.y.key]) + 12)
                .text(tooltip_text_y);
            })
            .on('mouseout', () => {
              this.tooltips_text.x.text('');
              this.tooltips_text.y.text('');
            });
        }
      },
      redraw_points: function(){
        select('.lineChartCompD3_all_points_g')
          .selectAll('.lineChartCompD3_points_g')
          .selectAll('.lineChartCompD3_points_text')
          .attr('x',d => {
            return this.scale.x(d[this.axis.x.key]);
          })
          .attr('y',d => {
            return this.scale.y(d[this.axis.y.key]);
          });
      },
      draw_axis_titles: function(){
        //draw x axis title
        const axis_title_g_x = this.chart_g.append('g')
          .attr('class', 'lineChartCompD3_axis_title_g_x');
        axis_title_g_x.append('text')
          .attr('class','lineChartCompD3_axis_title lineChartCompD3_axis_title_x')
          .attr('x', this.pixels.x / 3)
          .attr('y', this.pixels.y + this.margin_bottom - 10)
          .text(this.axis.x.title);
        make_draggable(axis_title_g_x);

        //draw y axis title
        const axis_title_g_y = this.chart_g.append('g')
          .attr('class', 'lineChartCompD3_axis_title_g_y');
        axis_title_g_y.append('text')
          .attr('class','lineChartCompD3_axis_title lineChartCompD3_axis_title_y')
          .attr('transform', 'rotate(-90)')
          .attr('x', -0.67 * this.pixels.y)
          .attr('y', -this.margin_left + 30)
          .text(this.axis.y.title);
        make_draggable(axis_title_g_y);
      },
      draw_main_titles: function(){
        //draw chart main titles
        const title_1_g = this.chart_g.append('g')
          .attr('class', 'lineChartCompD3_title_g_1');
        title_1_g.append('text')
          .attr('class','lineChartCompD3_title')
          .attr('transform', `translate(${this.margin_left + this.pixels.x / 2},30)`)
          .attr('text-anchor', 'middle')
          .style('font-size', '1.3rem')
          .text(this.title_1);
        make_draggable(title_1_g);

        const title_2_g = this.chart_g.append('g')
          .attr('class', 'lineChartCompD3_title_g_2');
        title_2_g.append('text')
          .attr('class','lineChartCompD3_title')
          .attr('transform', `translate(${this.margin_left + this.pixels.x / 2},50)`)
          .attr('text-anchor', 'middle')
          .style('font-size', '.8rem')
          .text(this.title_2);
        make_draggable(title_2_g);
      },
      draw_legend: function(){
        //draw legend -- check icons.length
        if(this.icons.length > 0){
          const legends_g = this.chart_g.append('g')
            .attr('class', 'lineChartCompD3_legends_g');
          const legend_g = legends_g.selectAll('g').data(this.icons).enter()
            .append('g')
            .attr('class', 'lineChartCompD3_legend_g')
            .attr('transform', (d, i) => {
              return `translate(${80 + i * 180},20)`;
            });
          legend_g.append('text')
            .attr('class', 'lineChartCompD3_legend_text')
            .text(d => {
              return `${d.text}:`;
            });
          legend_g.append('text')
            .attr('x', d => {
              return (d.text.length + 1) * 8;
            })
            .attr('fill', d => {
              return d.color;
            })
            .text(d => {
              return d.icon;
            });

          //if no icon for legend, then check for 'connect'
          for(let point of this.points){
            if(point.icon === undefined && point.connect === true){
              const legend_g = legends_g.selectAll('g').data(this.points).enter()
                .append('g')
                .attr('class', 'lineChartCompD3_legend_g')
                .attr('transform', (d, i) => {
                  return `translate(${80 + i * 180},20)`;
                });
              legend_g.append('text')
                .attr('class', 'lineChartCompD3_legend_text')
                .text(d => {
                  return `${d.text}:`;
                });
              legend_g.append('line')
                .attr('x1', d => {
                  return (d.text.length + 1) * 8;
                })
                .attr('y1', -2)
                .attr('x2', d => {
                  return (d.text.length + 1) * 8 + 30;
                })
                .attr('y2', -2)
                .attr('stroke', d => {
                  return d.color;
                })
                .attr('stroke-width', '1.5');
            }
          }
          //make legend draggable
          make_draggable(legends_g);
        }
      },
      draw_fit_lines: function(){
        if(this.fits !== null){
          const x_min = this.scale.x.domain()[0];
          const x_max = this.scale.x.domain()[1];

          //create the lines
          const all_fits_g = this.chart_g.append('g')
            .attr('class', 'lineChartCompD3_all_fits_g');
          all_fits_g.selectAll('path').data(this.fits).enter()
            .append('g')
            .attr('class', 'lineChartCompD3_fit_g')
            .append('path')
            .attr('class', 'lineChartCompD3_fit_path')
            .style('stroke', d => {
              return d.color;
            })
            .attr('d', d => {
              const regress_data = [];
              const x_step = (x_max - x_min) / d.number_of_points;
              for(let i = 0; i < d.number_of_points; i++){
                regress_data.push(d.equation(x_min + i * x_step));
              }
              return this.line_generator(regress_data);
            });
          //draw fits legend
          const fit_legends_g = this.chart_g.append('g')
            .attr('class', 'lineChartCompD3_fit_legends_g')
            .attr('transform', 'translate(80,46)');
          fit_legends_g.selectAll('text').data(this.fits).enter()
            .append('text')
            .attr('y', (d, i) => {
              return 24 * i;
            })
            .attr('fill', d => {
              return d.color;
            })
            .text((d => {
                return d.text;
              })
            );
          //make legend draggable
          make_draggable(fit_legends_g);
        }
      },
      redraw_fit_lines: function(){
        if(this.fits !== null){
          const x_min = this.scale.x.domain()[0];
          const x_max = this.scale.x.domain()[1];

          selectAll('.lineChartCompD3_all_fits_g')
            .selectAll('.lineChartCompD3_fit_g')
            .selectAll('.lineChartCompD3_fit_path')
            .attr('d', d => {
              const regress_data = [];
              const x_step = (x_max - x_min) / d.number_of_points;
              for(let i = 0; i < d.number_of_points; i++){
                regress_data.push(d.equation(x_min + i * x_step));
              }
              return this.line_generator(regress_data);
            });
        }
      },
      draw_connect_lines: function(){
        //connect the points?
        this.connected_points = [];
        for(let point of this.points){
          if(Boolean(point.connect) && point.connect){
            this.connected_points.push(point);
          }
        }
        if(this.connected_points.length > 0){
          const connections_g = this.chart_g.append('g')
            .attr('class', 'lineChartCompD3_connections_g');
          connections_g.selectAll('path').data(this.connected_points).enter()
            .append('path')
            .attr('class', (d,i) => {
              return 'lineChartCompD3_connect_path lineChartCompD3_connect_path_' + i;
            })
            .attr('d', (d,i) => {
              return this.line_generator_obj(this.grouped_data[i]);
            })
            .style('stroke', d => {
              return d.color;
            })
            .on('mouseover', (d,i) => {
              const connections_g = this.chart_g.select('.lineChartCompD3_connections_g');
              connections_g.selectAll('.lineChartCompD3_connect_path')
                .style('stroke', 'gray')
                .style('opacity', '.3');

              const connect_path = connections_g.select('.lineChartCompD3_connect_path_' + i);
              connect_path
                .style('stroke', d.color)
                .style('opacity', '1.0');
            })
            .on('mouseout', () => {
              const connections_g = this.chart_g.select('.lineChartCompD3_connections_g');
              connections_g.selectAll('.lineChartCompD3_connect_path')
                .style('stroke', d => {
                  return d.color;
                })
                .style('opacity', '1.0');
            });
        }
      },
      redraw_connect_lines: function(){
        this.chart_g.select('.lineChartCompD3_connections_g').selectAll('path')
          .attr('d',(d,i) => {
            return this.line_generator_obj(this.grouped_data[i]);
          });
      },
      create_tooltips: function(){
        this.tooltips_text = {};
        this.tooltips_text.x = this.chart_g.append('text')
          .attr('class', 'lineChartCompD3_tooltip lineChartCompD3_tooltip_x')
          .attr('x',0)
          .attr('y',0)
          .text('');
        this.tooltips_text.y = this.chart_g.append('text')
          .attr('class', 'lineChartCompD3_tooltip lineChartCompD3_tooltip_y')
          .attr('x',0)
          .attr('y',0)
          .text('');
      },
      set_initials: function(xy,min,max,tick_values,tick_format){
        this.initials[xy].min = min;
        this.initials[xy].max = max;
        this.initials[xy].tick_values = tick_values;
        this.initials[xy].tick_format = tick_format;
      },
      set_currents: function(xy,min,max,step,tick_format){
        if(this.axis[xy].data_type !== undefined && this.axis[xy].data_type === 'time'){
          this.current_min[xy] = timeFormat(this.axis[xy].tic_format)(min);
          this.current_max[xy] = timeFormat(this.axis[xy].tic_format)(max);
          this.current_step[xy] = step;
          this.current_tick_format[xy] = tick_format;
        }else {
          this.current_min[xy] = min;
          this.current_max[xy] = max;
          this.current_step[xy] = step;
          this.current_tick_format[xy] = tick_format;
        }

        //update gui
        if(this.current_axis === xy){
          this.min = this.current_min[xy];
          this.max = this.current_max[xy];
          this.step = this.current_step[xy];
        }
      },
      remove_elements: function(){
        //remove x axis
        this.chart_g.select('.lineChartCompD3_axis_x').remove();
        //remove y axis
        this.chart_g.select('.lineChartCompD3_axis_y').remove();
        //remove axis titles
        this.chart_g.select('.lineChartCompD3_axis_title_g_x').remove();
        this.chart_g.select('.lineChartCompD3_axis_title_g_y').remove();
        //remove main titles
        this.chart_g.select('.lineChartCompD3_title_g_1').remove();
        this.chart_g.select('.lineChartCompD3_title_g_2').remove();
        //remove points
        this.chart_g.select('.lineChartCompD3_all_points_g').remove();
        //remove line connections
        this.chart_g.select('.lineChartCompD3_connections_g').remove();
        //remove legend
        this.chart_g.select('.lineChartCompD3_legends_g').remove();
        //remove fit lines
        this.chart_g.select('.lineChartCompD3_all_fits_g').remove();
        //remove any fits legends
        this.chart_g.select('.lineChartCompD3_fit_legends_g').remove();
        //remove tooltips
        this.chart_g.select('.lineChartCompD3_tooltip_x').remove();
        this.chart_g.select('.lineChartCompD3_tooltip_y').remove();
      }
    },
    components: {
      ButtonComp,
      InputComp,
      SelectComp
    },
    mounted(){
      //locate svg
      const svg_class = this.$slots.default[0].elm.classList[0];
      const svg = select(`.${svg_class}`);

      const chart = init_chart(svg,this.margin_left,undefined,undefined,this.margin_bottom);
      this.pixels.x = chart.chart_width;
      this.pixels.y = chart.chart_height;
      this.chart_g = chart.chart_g;

      if(this.css_variables !== null){
        for(let key of Object.keys(this.css_variables)){
          this.$el.style.setProperty(`--${key}`, this.css_variables[key]);
        }
      }
    }
  }
</script>

<style lang="less">
  :root {
    --line_chart_compD3_font_family: Verdana,serif;
    --line_chart_compD3_color: black;
    --line_chart_compD3_background_color: white;

    --line_chart_compD3_axis_font_size: .6rem;
    --line_chart_compD3_axis_color: black;

    --line_chart_compD3_icon_opacity: .6;

    --line_chart_compD3_tooltip_fill: black;

    --line_chart_compD3_line_width: 1.5;
  }

  .buttonComp {
    align-self: center;
  }

  .lineChartCompD3 {
    display: flex;
    flex-direction: column;
    font-family: var(--line_chart_compD3_font_family);
    background-color: var(--line_chart_compD3_background_color);
    color: var(--line_chart_compD3_color);

    &_title {
      font-weight: bold;
      fill: var(--line_chart_compD3_color);
      cursor: move;
    }

    &_axis_title {
      font-size: 12px;
      font-weight: bold;
      fill: var(--line_chart_compD3_axis_color);
      cursor: move;
    }

    &_axis_x,
    &_axis_y {
      cursor: pointer;
    }

    &_axis path,
    &_axis line{
      shape-rendering: crispEdges;
      color: var(--line_chart_compD3_axis_color);
    }
    &_axis text {
      font-weight: bold;
      font-family: var(--line_chart_compD3_font_family);
      font-size: var(--line_chart_compD3_axis_font_size);
      color: var(--line_chart_compD3_axis_color);
    }

    &_points_text {
      text-anchor: middle;
      font-size: 12px;
      opacity: var(--line_chart_compD3_icon_opacity);
    }

    &_fit_path,
    &_connect_path {
      fill: none;
      stroke-width: var(--line_chart_compD3_line_width);
    }

    &_legend_text {
      font-size: 12px;
      fill: var(--line_chart_compD3_color);
    }

    &_equations_text {
      font-size: 12px;
    }

    &_scaling {
      display: flex;
      flex-direction: row;
      justify-content: space-between;
      align-items: center;
      width: 780px;
      margin-left: 20px;
      margin-bottom: 5px;
    }

    &_not_scaling {
      display: none;
    }

    &_tooltip {
      text-anchor: start;
      font-size: 11px;
      //font-weight: bold;
      font-family: var(--line_chart_compD3_font_family);
      fill: var(--line_chart_compD3_tooltip_fill)
    }

    &_legends_g,
    &_fit_legends_g{
      cursor: move;
    }
  }
</style>