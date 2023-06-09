
<template>
    <div :class="'MatcScatterPlot MatcScatterPlot' + mode  ">
       
        <div class="MatcScatterPlotfActionCntr" data-dojo-attach-point="actionCntr">
        </div>
        <div class="MatcScatterPlotCntr" data-dojo-attach-point="cntr" @click="onBackgroundClick">
            <span class="MatxAxisXLine MatcScatterPlotfScatternVisible MatcScatterPlotDurationHidden"
                data-dojo-attach-point="xLine"></span>
            <span class="MatxAxisYLine MatcScatterPlotfScatternVisible MatcScatterPlotDurationHidden"
                data-dojo-attach-point="yLine"></span>
            <div class="MatcScatterPlotCanvas" data-dojo-attach-point="canvas">
            </div>
            <span class="MatxAxisLabel xMaxLabel" data-dojo-attach-point="xMaxLabel"></span>
            <span class="MatxAxisLabel MatcScatterPlotDropOffHidden yMaxLabel"
                data-dojo-attach-point="yMaxLabel"></span>
            <span class="MatxAxisLabel MatcScatterPlotfScatternVisible MatcScatterPlotDurationHidden xMiddleLabel"
                data-dojo-attach-point="xMiddleLabel"></span>
            <span class="MatxAxisLabel MatcScatterPlotfScatternVisible MatcScatterPlotDurationHidden yMiddleLabel"
                data-dojo-attach-point="yMiddleLabel"></span>
            <span class="MatxAxisLabel minLabel" data-dojo-attach-point="minLabel">0</span>
            <span class="MatxAxisLabel xLabel " data-dojo-attach-point="xLabel">X</span>
            <span class="MatxAxisLabel yLabel MatcScatterPlotDropOffHidden" data-dojo-attach-point="yLabel">Y</span>

            <span class="MatxAxisLabel yLabelEast MatcScatterPlotfDetailsVisible"
                data-dojo-attach-point="yLabelEast">Y</span>
            <span class="MatxAxisLabel yMinLabelEast MatcScatterPlotfDetailsVisible"
                data-dojo-attach-point="yMinLabelEast">0</span>
            <span class="MatxAxisLabel yMaxLabelEast MatcScatterPlotfDetailsVisible"
                data-dojo-attach-point="yMaxLabelEast">100</span>
            <span class="MatxAxisLabel yMinLabel" data-dojo-attach-point="yMinLabel"></span>

            <span class="MatxAxisLabel bottom25Label MatcScatterPlotfDetailsVisible"
                data-dojo-attach-point="bottom25Label"></span>
            <span class="MatxAxisLabel bottom75Label MatcScatterPlotfDetailsVisible"
                data-dojo-attach-point="bottom75Label"></span>

            <span class="MatxAxisLabel MatxAxisLabelCntr" data-dojo-attach-point="xAxisLabelCntr"></span>


            <div class="MatcScatterPlotfTaskCntr" data-dojo-attach-point="taskCntr">
            </div>



        </div>

        <div class="MatcScatterPlotHintBar" data-dojo-attach-point="hintCntr">
        </div>

    </div>
</template>

<style lang="scss">
    @import '../../../style/scatter.scss';
</style>

<script>
import on from 'dojo/on'
import lang from 'dojo/_base/lang'
import css from 'dojo/css'
import Logger from 'common/Logger'
import _Color from 'common/_Color'
import DomBuilder from 'common/DomBuilder'
import Util from 'core/Util'
import Analytics from 'dash/Analytics'
import DataFrame from 'common/DataFrame'



export default {
    name: 'ScatterPlot',
    mixins: [_Color, Util],
    props: ['test', 'app', 'events', 'annotation', 'pub'],
    data: function () {
        return {
            mode: 'Scatter',   
            paddingFactor: 1.1,
            dialog: false,
            includeDropOff: false,
            defaultColor: "#56A9FC",
            colors: [ "#9933cc", "#669900", "#ff8a00", "#cc0000", "#000000", "#8ad5f0", "#d6adeb", "#c5e26d"],
            bins: 3,
            canvasPos: {
                w: 800,
                h: 450
            }
        }
    },
    components: {

    },
    methods: {
        postCreate() {
            this.log = new Logger("ScatterPlot");
            this.init();
            this._scatterPoints = {}
        },

        init() {
            this.db = new DomBuilder();      
            this._scatterPoints = {};
        },

        setValue(test, app, events, annotations) {

            const filteredEvents = this.filterEvents(events, annotations);
            const df = this.getActionEvents(new DataFrame(filteredEvents));
            df.sortBy("time");

            this.model = app
            this.df = df

            this.annotations = annotations;
            this.tasks = lang.clone(test.tasks).filter(task => task.flow.length >= 2);
            if (this.scatterMode === 'task') {
                this.tasks.unshift({
                    name: 'All',
                    id: '_all',
                    isAll: true,
                    flow: []
                })       
            }

            const analytics = new Analytics();
            const taskPerformance = analytics.getTaskPerformance(df, this.tasks);

            const sessionSummary = this.getSessionSummary(df, this.tasks[0])
            this.sessionSummary = sessionSummary
            this.taskPerformance = taskPerformance.merge(sessionSummary)

            this.renderTasks(this.tasks);
            if (this.scatterMode === 'task') {
                this.selectTask(this.tasks[0]);
            }
            this.render();
        },

        getSessionSummary (df, taskName = 'All', taskID = -1) {
            const result = []
            const sessions = df.groupBy("session");
            sessions.foreach((session, id) => {
                result.push({
                    interactions: session.size(),
                    session: id, 
                    task: taskID,
                    taskName: taskName,
                    duration:  Math.ceil((session.max("time") - session.min("time")))
                })
            })
            return new DataFrame(result);
	    },

        renderTasks(tasks) {
            this.log.log(1, "renderTasks", "enter");
            this.taskDivs = {};
            this.taskCircles = {};
            this.selectedTasks = {};
            this.taskColors = {};        
            for (let i = 0; i < tasks.length; i++) {
                const task = tasks[i];
                if (task.flow.length >= 2 ) {
                    this.renderTaskBubble(task, i)
                }
            }
        },

        renderTaskBubble(task, i ) {
            this.selectedTasks[task.id] = false;
            this.taskColors[task.id] = this.colors[i % this.colors.length];

            const div = this.db.div().build(this.taskCntr);
            const circle = this.db.span("").build(div);

            this.db.label("", task.name).build(div);
            this.taskCircles[task.id] = circle;
            this.taskDivs[task.id] = div;

            this.own(on(div, "click", lang.hitch(this, "clickTask", task, i)));
        },

        clickTask(task) {
            this.selectTask(task);
            this.render(true);
        },

        selectTask(task) {
            this.selectedTasks[task.id] = !this.selectedTasks[task.id];
            for (let id in this.selectedTasks) {
                if (id !== task.id) {
                    this.selectedTasks[id] = false
                }
            }
      
            for (let id in this.selectedTasks) {
                const circle = this.taskCircles[id];
                const div = this.taskDivs[id];
                if (div) {
                    if (this.selectedTasks[id]) {
                        css.add(div, "MatcScatterPlotfTaskSelected");
                        circle.style.background = this.taskColors[id];                  
                    } else {
                        css.remove(div, "MatcScatterPlotfTaskSelected");
                        circle.style.background = "";
                    }
                }
            }

        },

        render(changeTask) {
            this.log.log(-1, "render", "enter > ");
            this.cleanUpTempListener()           
            try {
                if (!changeTask && this["clean_" + this.lastMode]) {
                this["clean_" + this.lastMode](lang.hitch(this, "renderTab", changeTask));
                } else {
                    this.renderTab(changeTask);
                }      
            } catch (err) {
                this.log.error("render", "error > ");
                this.log.sendError(err)
            }
                
        },

        renderTab(changeTask) {
            this.log.log(-1, "renderTab", "enter > " + this.mode);
            if (this["render_" + this.mode]) {
                this["render_" + this.mode](this.df, this.task, this.annotations, this.tasks, changeTask);
            } else {
                this.log.error("render", "No render for " + this.tab);
            }
        },

        onBackgroundClick () {
            this.setHint(this.db.span("MatcHint", this.getNLS("analytics.scatter.hint")).build());
        },


        /*********************************************************************
         * BoxPlot
         *********************************************************************/


        render_BoxPlot() {
            this.log.log(-1, "render_duration", "enter > changeTask:");
            const perf = this.taskPerformance

            css.add(this.domNode, "MatcScatterPlotDetails");
            this.xLabel.innerHTML = ""; //this.getNLS("dash.perf.details.xLabel");
            this.yLabel.innerHTML = "";//this.getNLS("dash.perf.details.yLabel");
            this.yLabelEast.innerHTML = ""; //this.getNLS("dash.perf.details.yLabelEast");
            this.bottom25Label.innerHTML = this.getNLS("dash.perf.details.bottom25Label");
            this.bottom75Label.innerHTML = this.getNLS("dash.perf.details.bottom75Label");
            this.xMaxLabel.innerHTML = "";
            this.yMinLabel.innerHTML = ""
            this.minLabel.innerHTML = ""

            // loop to get the max
            var max_duration = 1;
            var max_count = 1;
            var selectCount = 0;
            for (let id in this.selectedTasks) {
                if (this.selectedTasks[id]) {
                    let taskDF = perf.select("task", "==", id);
                    max_duration = Math.max(max_duration, Math.ceil(taskDF.max("duration") * this.paddingFactor));
                    max_count = Math.max(max_count, Math.ceil(taskDF.max("interactions") * this.paddingFactor));
                    selectCount++;
                }
            }

            this.yMaxLabel.innerHTML = max_count;
            this.yMaxLabelEast.innerHTML = Math.ceil(max_duration / 1000) + " s";


            /**
             * Now render selected tasks and remove not selected tasks
             */
            var i = 0;
            var w = 2;
            var slots = (selectCount * 2) - 1;
            for (let id in this.selectedTasks) {
                if (this.selectedTasks[id]) {
                    let taskDF = perf.select("task", "==", id);

                    /**
                     * Create interactions box plot
                     */
                    let max = Math.ceil(taskDF.max("interactions"));
                    let min = Math.ceil(taskDF.min("interactions"));
                    let mean = Math.ceil(taskDF.mean("interactions"));
                    let std = Math.ceil(taskDF.std("interactions"));
                    let left = (25 - ((w * slots) / 2)) + (i * 2 * w) + "%";
                    this.createBoxPlot(id, "i", i, left, w, max, min, mean, std, max_count, this.taskColors[id]);

                    /**
                     * Create duration box plot
                     */
                    max = Math.ceil(taskDF.max("duration"));
                    min = Math.ceil(taskDF.min("duration"));
                    mean = Math.ceil(taskDF.mean("duration"));
                    std = Math.ceil(taskDF.std("duration"));
                    left = (75 - ((w * slots) / 2)) + (i * 2 * w) + "%";
                    this.createBoxPlot(id, "d", i, left, w, max, min, mean, std, max_duration, this.taskColors[id]);

                    i++;
                } else {
                    /**
                     * Remove not needed box plots
                     */
                    if (this._bars) {
                        if (this._bars[id + "i"]) {
                            let bar = this._bars[id + "i"];
                            bar.parentNode.removeChild(bar);
                            delete this._bars[id + "i"]
                        }
                        if (this._bars[id + "d"]) {
                            let bar = this._bars[id + "d"];
                            bar.parentNode.removeChild(bar);
                            delete this._bars[id + "d"]
                        }
                    }
                }
            }
        },

        createBoxPlot(id, prefix, i, left, width, max, min, mean, std, total, color, widthUnit = '%') {

            if (!this._bars) {
                this._bars = {}
            }

            var height = (((max - min) / total) * 100) + "%";
            var ms = Math.min(i * 60, 500);
            if (!this._bars[id + prefix]) {
                /**
                 * Create new
                 */
                let cntr = this.db.div("MatcScatterPlotBoxPlotCntr").build(this.canvas);
                cntr.style.bottom = ((min / total) * 100) + "%"
                cntr.style.left = left;
                cntr.style.height = "0px";
                cntr.style.width = width + widthUnit

                this.db.div("MatcScatterPlotBoxPlotLine").build(cntr);

                var bar = this.db.div("MatcScatterPlotBoxPlot").build(cntr);
                bar.style.bottom = Math.max(0, ((((mean - std) - min) / (max - min)))) * 100 + "%";
                bar.style.height = ((std * 2) / (max - min)) * 100 + "%";
                bar.style.background = color

                var centre = this.db.div("MatcScatterPlotBoxPlotMean").build(cntr);
                centre.style.bottom = (((mean - min) / (max - min))) * 100 + "%"

                setTimeout(lang.hitch(this, "animateBoxplot", cntr, height), ms);
                this._bars[id + prefix] = cntr;

            } else {
                /**
                 * Update exiting ones
                 */
                let cntr = this._bars[id + prefix];
                cntr.style.left = left;
                setTimeout(lang.hitch(this, "animateBoxplot", cntr, height), ms);
            }
        },

        animateBoxplot(bar, height) {
            bar.style.height = height;
        },

        clean_BoxPlot(callback) {
            this.log.log(-1, "clean_duration", "enter");
            css.remove(this.domNode, "MatcScatterPlotDetails");
            this.setHint();

            if (this._bars) {
                for (var id in this._bars) {
                    var b = this._bars[id];
                    b.style.height = "0%";
                    b.style.bottom = "0%";
                }
            }
            setTimeout(lang.hitch(this, "removeBoxplots"), 200);

            if (callback) {
                setTimeout(callback, 200);
            }
        },


        removeBoxplots  () {
            if (this._bars) {
                for (var id in this._bars) {
                    var b = this._bars[id];
                    this.canvas.removeChild(b)
                }
            }
            delete this._bars;
        },



        /*********************************************************************
         * Scatter
         *********************************************************************/

     

        render_Scatter() {
            this.log.log(-1, "render_Scatter", "enter > changeTask:" + this.scatterMode);

            delete this._selectedScatterPoint;
            this.onBackgroundClick()

            css.add(this.domNode, "MatcScatterPlotScatter");
            this.xLabel.innerHTML = this.getNLS("dash.perf.scatter.xLabel");
            this.yLabel.innerHTML = this.getNLS("dash.perf.scatter.yLabel");
            this.minLabel.innerHTML = "0 s"
            this.yMinLabel.innerHTML = "0"

            if (this.scatterMode === 'task') {
                this.render_Scatter_Tasks()
            } else {
                this.render_Scatter_Session()
            }
        },

        render_Scatter_Session () {
            this.log.log(-1, "render_Scatter_Tasks", "enter" );

            const db = new DomBuilder();
            const sessionSummaryDF = this.sessionSummary
            const perf = this.taskPerformance
            let mean_duration = 0
            let mean_count = 0
            let maxDelay = 0;

        
        
            let max_duration = Math.max(1, Math.ceil(sessionSummaryDF.max("duration") * this.paddingFactor));
            let max_count = Math.max(1, Math.ceil(sessionSummaryDF.max("interactions") * this.paddingFactor));
       
            this.xMaxLabel.innerHTML = Math.ceil(max_duration / 1000) + " s";
            this.yMaxLabel.innerHTML = Math.ceil(max_count);
     
            const sessions = sessionSummaryDF.data;
            sessions.sort((a, b) => {
                return b.interactions - a.interactions;
            });

            
            mean_duration = sessionSummaryDF.mean("duration");
            mean_count = sessionSummaryDF.mean("interactions");

            for (let i = 0; i < sessions.length; i++) {
                const s = sessions[i]
                const key = s.session;
                const ms = Math.min(i * 10, 300);
                maxDelay = Math.max(maxDelay, ms)

                if (this._scatterPoints[key]) {
                    const p = this._scatterPoints[key];
                    p.style.background = this.defaultColor
                    this.tempOwn(on(p, "click", lang.hitch(this, "selectPoint", p, s, i)));        
                    setTimeout(lang.hitch(this, "animateScatterPoint", p, s, max_duration, max_count), ms);
                } else {
                    const p = db.span("MatxScatterPoint").build(this.canvas);
                    p.style.bottom = "-5%";
                    p.style.left = (((s.duration / max_duration) * 100)) + "%";
                    p.style.background = this.defaultColor
                    this._scatterPoints[key] = p;
                    this.tempOwn(on(p, "click", lang.hitch(this, "selectPoint", p, s, i)));
            
                    setTimeout(lang.hitch(this, "animateScatterPoint", p, s, max_duration, max_count), ms);
                }
            }

            for (let id in this.selectedTasks) {
                const taskDF = perf.select("task", "==", id);
                const sessions = taskDF.data;
                sessions.sort((a, b) => {
                    return b.interactions - a.interactions;
                });
                if (this.selectedTasks[id]) {
                    for (let i = 0; i < sessions.length; i++) {
                        const s = sessions[i];          
                        const key = s.session;
                        const p = this._scatterPoints[key];
                        p.style.background = this.taskColors[id];
                    }
                }
            }
                
            this.max_interactions = max_count;
            this.max_duration = max_duration;

            setTimeout(lang.hitch(this, "setMiddle", mean_count, max_count, mean_duration, max_duration), 200);
        },

        render_Scatter_Tasks () {
            this.log.log(-1, "render_Scatter_Tasks", "enter" );
            const perf = this.taskPerformance
            const db = new DomBuilder();

 
            // loop to get the max
            var max_duration = 1;
            var max_count = 1;
            for (let id in this.selectedTasks) {
                if (this.selectedTasks[id]) {
                    let taskDF = perf.select("task", "==", id);
                    max_duration = Math.max(max_duration, Math.ceil(taskDF.max("duration") * this.paddingFactor));
                    max_count = Math.max(max_count, Math.ceil(taskDF.max("interactions") * this.paddingFactor));
                }
            }


            this.xMaxLabel.innerHTML = Math.ceil(max_duration / 1000) + " s";
            this.yMaxLabel.innerHTML = Math.ceil(max_count);

            let mean_duration = 0
            let mean_count = 0
            for (let id in this.selectedTasks) {
                const taskDF = perf.select("task", "==", id);
                const sessions = taskDF.data;
                sessions.sort((a, b) => {
                    return b.interactions - a.interactions;
                });

                if (this.selectedTasks[id]) {
                    mean_duration = taskDF.mean("duration");
                    mean_count = taskDF.mean("interactions");
                    var maxDelay = 0;
                    for (let i = 0; i < sessions.length; i++) {
                        const s = sessions[i];
                        const key = s.session + " " + id;
                        const ms = Math.min(i * 10, 300);
                        maxDelay = Math.max(maxDelay, ms)

                        if (this._scatterPoints[key]) {
                            const p = this._scatterPoints[key];
                            this.tempOwn(on(p, "click", lang.hitch(this, "selectPoint", p, s, i)));
               
                            setTimeout(lang.hitch(this, "animateScatterPoint", p, s, max_duration, max_count), ms);
                        } else {
                            const p = db.span("MatxScatterPoint").build(this.canvas);
                            p.style.bottom = "-5%";
                            p.style.left = (((s.duration / max_duration) * 100)) + "%";
                            p.style.background = this.taskColors[id];
                            this._scatterPoints[key] = p;
                            this.tempOwn(on(p, "click", lang.hitch(this, "selectPoint", p, s, i)));
                   
                            setTimeout(lang.hitch(this, "animateScatterPoint", p, s, max_duration, max_count), ms);
                        }
                    }
                } else {
                    // remove all not needed elements
                    for (let i = 0; i < sessions.length; i++) {
                        let s = sessions[i];
                        let key = s.session + " " + id;
                        if (this._scatterPoints[key]) {
                            let p = this._scatterPoints[key];
                            p.style.opacity = 0;
                            setTimeout(lang.hitch(this, "fadeOutPoint", p), 300);
                            delete this._scatterPoints[key]
                        }
                    }
                }
            }

            // save values for select events
            this.max_interactions = max_count;
            this.max_duration = max_duration;

            // FIXME: Should be per task...
            setTimeout(lang.hitch(this, "setMiddle", mean_count, max_count, mean_duration, max_duration), 200);
        },

        fadeOutPoint(p) {
            if (p.parentNode) {
                p.parentNode.removeChild(p);
            }
        },

        clean_scatter(callback) {
            this.log.log(-1, "clean_scatter", "enter");
            css.remove(this.domNode, "MatcScatterPlotScatter");
            this.setHint();
            if (this._scatterPoints) {
                for (var sessionID in this._scatterPoints) {
                    var p = this._scatterPoints[sessionID];
                    p.style.bottom = "-5%";
                }
            }
            setTimeout(lang.hitch(this, "removePoints"), 200);
            if (callback) {
                setTimeout(callback, 200);
            }
            this.yMinLabel.innerHTML = ""
            this.minLabel.innerHTML = ""
        },

        removePoints() {
            if (this._scatterPoints) {
                for (var sessionID in this._scatterPoints) {
                    var p = this._scatterPoints[sessionID];
                    if (p.parentNode) {
                        p.parentNode.removeChild(p)
                    }

                }
            }
            delete this._scatterPoints;
            this._scatterPoints = {};
        },

        selectPoint(p, s, i, e) { 
            this.stopEvent(e);
            this.setXMiddle(s.duration, this.max_duration);
            this.setYMiddle(s.interactions, this.max_interactions);
            css.add(this.cntr, "MatcScatterPlotCntrHover");
            this._selectedScatterPoint = p;
            this.showSessionReplayHint(s.session)
        },

        showSessionReplayHint(id) {
            if (this.model) {
                let url = "#/apps/" + this.model.id + "/replay/" + id + ".html";
                if (this.mode == "public") {
                    url = "#/examples/" + this.model.id + "/replay/" + id + ".html";
                }
                const hint = this.db.span("MatcHint", this.getNLS("analytics.scatter.play")).build();
                const a = this.db.a("", this.getNLS("analytics.scatter.here")).build(hint);
                a.href = url
                a.target = "_matcSessionReplay" + id
                this.setHint(hint);
            }
        },

        /*********************************************************************
         * Helper
         *********************************************************************/

        hoverPoint(p) {
            if (this._selectedScatterPoint == p) {
                css.add(this.cntr, "MatcScatterPlotCntrHover");
            }
        },

        setHint(hintNode) {
            this.hintCntr.innerHTML = "";
            if (hintNode) {
                this.hintCntr.appendChild(hintNode);
            }
        },

        clearPoint() {
            //this.setMiddle(mean_count, max_count, mean_duration, max_duration);
            css.remove(this.cntr, "MatcScatterPlotCntrHover");
        },

        setMiddle(mean_count, max_count, mean_duration, max_duration) {
            this.setXMiddle(mean_duration, max_duration);
            this.setYMiddle(mean_count, max_count);
            css.remove(this.cntr, "MatcScatterPlotCntrHover");
        },

        setYMiddle(count, max_count) {
            this.yMiddleLabel.innerHTML = Math.ceil(count);
            this.yMiddleLabel.style.bottom = ((count / max_count) * 100) + "%";
            this.yLine.style.bottom = ((count / max_count) * 100) + "%";
        },

        setXMiddle(duration, max_duration) {
            this.xMiddleLabel.innerHTML = Math.ceil(duration / 1000) + " s";
            this.xMiddleLabel.style.left = (((duration / max_duration) * 100)) + "%";
            this.xLine.style.left = (((duration / max_duration) * 100)) + "%";
        },

        animateScatterPoint(p, s, max_duration, max_interactions) {
            p.style.bottom = (((s.interactions / max_interactions) * 100)) + "%";
            p.style.left = (((s.duration / max_duration) * 100)) + "%";
        }
    },
    mounted() {        
        this.setValue(this.test, this.app, this.events, this.annotation)
    }
}
</script>