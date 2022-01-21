<template>
  <div :class="rootClass" v-resize="resize" id="heb">
  </div>
</template>
<script>
import resize from 'vue-resize-directive'
import layout from './layout/circular'
import {compareNode, roundPath, toPromise, translate, updateTexts} from './d3-utils'

import * as d3 from 'd3'

const props = {
  data: Object,
  links: Array,
  marginX: {
    type: Number,
    default: 0
  },
  rootClass: {
    type: String,
    default: 'graph'
  },
  marginY: {
    type: Number,
    default: 0
  },
  maxTextWidth: {
    type: Number,
    default: -1
  },
  nodeText: {
    type: String,
    required: true
  },
  identifier: {
    required: true,
    validator (value) {
      const valueType = typeof value
      return (valueType === 'string') || (valueType === 'function')
    }
  },
  duration: {
    type: Number,
    default: 500
  },
  arcSpacing: {
    type: Number,
    default: 3
  }
}

const directives = {
  resize
}

export default {
  name: 'HierarchicalEdgeBundling',

  props,

  directives,

  data () {
    return {
      textContraint: null,
      highlightedNode: null,
      highlightedArc: null,
      m0: null,
      rotate: 0,
      tooltip: null
    }
  },

  mounted () {
    const size = this.getSize()
    const svg = d3.select(this.$el).append('svg')
          .attr('width', size.width)
          .attr('height', size.height)
    const g = this.transformSvg(svg.append('g'), size)
    console.log(size)
    this.tooltip = d3.select(this.$el)
      .append('div')
      .style('opacity', 0)
      .attr('class', 'tooltip')
      .style('background-color', 'white')
      .style('border', 'solid')
      .style('border-width', '2px')
      .style('border-radius', '5px')
      .style('padding', '5px')

    g.append('svg:path')
      .attr('class', 'arc')
      .attr('d', d3.arc().outerRadius(this.$el.clientHeight / 2).innerRadius(0).startAngle(0).endAngle(2 * Math.PI))
      .on('mousedown', this.mouseDown)

    d3.select(window)
      .on('mousemove', this.mouseMove)
      .on('mouseup', this.mouseUp)

    const tree = this.tree

    this.internaldata = {
      svg,
      g,
      tree
    }

    if (!this.data) {
      return
    }
    this.onData(this.data)
    this.links && this.onLinks(this.links)
    this.onCategories(this.categories)

    this.$el.addEventListener('click', this.handleClickOutside, true)
  },

  beforeDestroy () {
    this.$el.removeEventListener('click', this.handleClickOutside, true)
  },

  methods: {
    getSize () {
      var width = this.$el.clientWidth
      var height = this.$el.clientHeight
      return { width, height }
    },

    resize () {
      const size = this.getSize()
      const {g, svg, tree} = this.internaldata
      svg.attr('width', size.width)
        .attr('height', size.height)
      this.transformSvg(g, size)
      layout.optimizeSize(tree, size, this.margin, this.textContraint)
      this.redraw()
    },

    handleClickOutside (ev) {
      const el = this.internaldata.svg.node()
      if ((el === ev.target) || (!el.contains(ev.target))) {
        this.$emit('clickOutsideGraph')
      }
    },

    completeRedraw ({margin = null}) {
      const size = this.getSize()
      layout.optimizeSize(this.internaldata.tree, size, this.margin, this.textContraint)
      this.applyTransition(size, {margin})
      this.redraw()
    },

    transformSvg (g, size) {
      size = size || this.getSize()
      return layout.transformSvg(g, this.margin, size)
    },

    updateTransform (g, size) {
      size = size || this.getSize()
      return layout.updateTransform(g, this.margin, size)
    },

    updateNodes () {
      const {root, g, tree} = this.internaldata

      tree(root)
      const node = g.selectAll('.nodetree').data(root.leaves(), d => d._id)
      const newNodes = node.enter().append('g').attr('class', 'nodetree')
      newNodes.on('mouseover', this.mouseOvered).on('mouseout', this.mouseOuted).on('click', this.nodeClick)

      const allNodes = this.internaldata.nodes = newNodes.merge(node)
      const allNodesPromise = toPromise(allNodes.transition().duration(this.duration).attr('transform', d => translate(d, layout)).attr('opacity', 1))

      const {layoutNode, transformNode} = layout
      allNodes.each((d) => {
        d.layoutInfo = layoutNode(false, 6, d)
      })

      // newNodes.append('svg:a').attr('xlink:href', function (d) { return '/application/' + d._id }).append('text').attr('dy', '.35em')
      newNodes.append('text').attr('dy', '.35em')

      const text = allNodes.select('text')
        .text(d => d.data[this.nodeText])
        .attr('x', d => d.layoutInfo.x)
        .call(updateTexts, this.maxTextWidth)
        .attr('text-anchor', d => d.layoutInfo.anchor)
        .attr('transform', d => `rotate(${d.layoutInfo.rotate + d.layoutInfo.textRotate})`)

      const tentative = []
      text.each(function (d) { tentative.push({ node: this, data: d, pos: transformNode(d.x, this.getComputedTextLength() + 6) }) })

      const getMaxNode = (position) => {
        const mapped1 = tentative.map(el => ({ node: el.node, x: el.data.x, value: Math.abs(el.pos[position]) }))
        const max = Math.max(...mapped1.map(el => el.value))
        return mapped1.find(el => el.value === max)
      }
      const xExtreme = getMaxNode('x')
      const yExtreme = getMaxNode('y')
      const textContraint = {xExtreme, yExtreme}

      if ((this.textContraint) && (this.textContraint.xExtreme.value === xExtreme.value) &&
            (this.textContraint.yExtreme.value === yExtreme.value)) {
        return allNodesPromise
      }

      this.instantClean()
      this.textContraint = textContraint
      const size = this.getSize()
      this.transformSvg(g, size)
      layout.optimizeSize(tree, size, this.margin, this.textContraint)
      return this.updateNodes()
    },

    updateLinks () {
      const {g, links} = this.internaldata
      if (!links) {
        return
      }
      const edges = g.selectAll('.link').data(links, l => l.source._id + '-' + l.target._id + '-' + l.type)
      const line = layout.getLine(d3).curve(d3.curveBundle.beta(0.55))

      const newEdges = edges.enter().append('path').attr('class', 'link')
                            .attr('d', d => roundPath(line(d.source.path(d.target).map(p => ({x: p.x, y: 0.1})))))

      const allEdges = this.internaldata.edges = edges.merge(newEdges)
      const promise = toPromise(allEdges.transition().duration(this.duration).attr('d', d => roundPath(line(d.source.path(d.target)))))

      edges.exit().remove()
      return promise
    },

    updateCategories () {
      const {g, categories} = this.internaldata
      if (!categories) {
        return
      }

      const arcs = g.selectAll('.category').data(categories)

      const newArcs = arcs.enter().append('g')
        .attr('class', 'category')
        .on('mouseover', this.arcMouseover)
        .on('mousemove', this.arcMousemove)
        .on('mouseleave', this.arcMouseleave)

      newArcs.append('svg:path')
        .attr('id', function (d, i) { return 'categoryArc_' + i })
        .attr('d', function (d) {
          return d3.arc()({
            outerRadius: d.r - 5,
            innerRadius: d.r - 35,
            startAngle: (d.start / 180) * Math.PI,
            endAngle: (d.end / 180) * Math.PI
          })
        })
        .style('fill', function (d) { return d.color })

      newArcs.append('svg:text')
        .attr('class', 'categoryText')
        .attr('x', 5) // Move the text from the start angle of the arc
        .attr('dy', 20) // Move the text down
        .append('svg:textPath')
        .attr('xlink:href', function (d, i) { return '#categoryArc_' + i })
        .text(function (d) { return d.displayName })

      const allArcs = this.internaldata.arcs = arcs.merge(newArcs)
      const promise = toPromise(allArcs.transition().duration(this.duration * 2).attr('opacity', 1))

      arcs.exit().remove()
      return promise
    },

    mouseOvered (d) {
      this.emit('mouseNodeOver', d)
    },

    mouseOuted (d) {
      this.emit('mouseNodeOut', d)
    },

    nodeClick (d) {
      console.log('mouse clicked')
      this.emit('mouseNodeClick', d)
    },

    arcMouseover (d) {
      this.tooltip.style('opacity', 1)
    },

    arcMousemove (d) {
      console.log(d)
      console.log(d3.event)
      this.tooltip
      .html(d.name)
      .style('left', (d3.event.offsetX + 10) + 'px')
      .style('top', (d3.event.offsetY + 10) + 'px')
    },

    arcMouseleave (d) {
      this.tooltip
        .style('opacity', 0)
    },

    mouse (e) {
      return [e.pageX - this.$el.clientWidth / 2, e.pageY - this.$el.clientHeight / 2]
    },

    mouseDown () {
      this.m0 = this.mouse(d3.event)
      d3.event.preventDefault()
    },

    mouseMove () {
      d3.event.preventDefault()
      if (this.m0) {
        var m1 = this.mouse(d3.event)
        var dm = Math.atan2(this.cross(this.m0, m1), this.dot(this.m0, m1)) * 180 / Math.PI
        d3.select(this.$el).style('-webkit-transform', 'translateY(0px)rotateZ(' + dm + 'deg)translateY(0px)')
      }
    },

    mouseUp () {
      if (this.m0) {
        const {g, nodes} = this.internaldata
        var m1 = this.mouse(d3.event)
        var dm = Math.atan2(this.cross(this.m0, m1), this.dot(this.m0, m1)) * 180 / Math.PI

        this.rotate += dm
        if (this.rotate > 360) this.rotate -= 360
        else if (this.rotate < 0) this.rotate += 360
        this.m0 = null

        d3.select(this.$el).style('-webkit-transform', null)
        const size = this.getSize()
        let that = this
        // console.log(this.rotate)
        g.attr('transform', 'translate(' + size.width / 2 + ',' + size.height / 2 + ')rotate(' + this.rotate + ')')
          .selectAll('g.nodetree text')
          .attr('text-anchor', function (d) {
            // console.log(d)
            // console.log((d.layoutInfo.rotate + that.rotate + 90) % 360)
            return (d.layoutInfo.rotate + that.rotate + 90) % 360 < 180 ? 'start' : 'end'
          })
          .attr('transform', function (d) { return (d.layoutInfo.rotate + that.rotate + 90) % 360 < 180 ? `rotate(${d.layoutInfo.rotate})` : `rotate(${d.layoutInfo.rotate + 180} )` })
        nodes.each(n => {
          n.layoutInfo.anchor = (n.layoutInfo.rotate + that.rotate + 90) % 360 < 180 ? 'start' : 'end'
          n.layoutInfo.textRotate = (n.layoutInfo.rotate + that.rotate + 90) % 360 < 180 ? 0 : 180
        })
        // console.log(nodes)
      }
    },

    cross (a, b) {
      return a[0] * b[1] - a[1] * b[0]
    },

    dot (a, b) {
      return a[0] * b[0] + a[1] * b[1]
    },

    emit (name, d) {
      this.$emit(name, {element: d, data: d.data})
    },

    showDependencies (d) {
      const {edges, nodes, arcs} = this.internaldata
      if (!edges) {
        return
      }

      console.log('arcs')
      console.log(arcs)
      console.log(edges)
      nodes.each(n => { n.target = n.source = false })

      const rootElement = d3.selectAll([this.$el]).style('display', 'none').classed('detailed', true)

      edges.filter(l => l.target === d || l.source === d)
      .classed('link--target', function (l) {
        if (l.target === d) {
          l.source.source = true
          return true
        }
      })
      .classed('link--source', function (l) {
        if (l.source === d) {
          l.target.target = true
          return true
        }
      })
      .raise()

      const nodesSelected = nodes.filter(n => ((n.target) || (n.source) || (n === d)))
        .classed('node--target', n => n.target)
        .classed('node--source', n => n.source)
        .classed('node--selected', n => n === d)

      rootElement.style('display', 'block')
      nodesSelected
        .select('text')
        .attr('text-anchor', d => d.layoutInfo.anchor)
      console.log(d)
      arcs.filter(l => l.name === d.parent.data.text)
        .classed('parent-arc', true)
        .raise()
    },

    reset (d) {
      const {edges, nodes, arcs} = this.internaldata
      if (!edges) {
        return
      }
      const rootElement = d3.selectAll([this.$el]).style('display', 'none').classed('detailed', false)

      arcs.classed('parent-arc', false)

      edges.classed('link--target', false)
          .classed('link--source', false)

      nodes.classed('node--target', false)
          .classed('node--source', false)
          .classed('node--selected', false)

      nodes.filter(n => ((n.target) || (n.source) || (n === d)))
          .select('text').attr('dx', d => d.layoutInfo.standardDx)

      rootElement.style('display', 'block')
    },

    onData (data) {
      if (!data) {
        this.internaldata.root = this.internaldata.nodes = null
        return
      }
      const root = d3.hierarchy(data).sort((a, b) => compareNode(a, b, this.nodeText))
      this.internaldata.root = root
      this.$emit('nodesComputed', root)
      const map = this.internaldata.map = {}
      const identifier = this.identifier
      const idGetter = typeof identifier === 'string' ? data => data[identifier] : identifier
      root.each(d => {
        const id = idGetter(d.data)
        d._id = id
        map[id] = d
      })
      const size = this.getSize()
      root.x = size.height / 2
      root.y = 0
      root.x0 = root.x
      root.y0 = root.y
      this.updateNodes()
      this.onCategories()
    },

    onLinks (links) {
      if (!this.data) {
        return
      }

      if (!links) {
        this.internaldata.links = this.internaldata.edges = null
      }

      const {map} = this.internaldata
      this.internaldata.links = links.map(link => ({source: map[link.source], target: map[link.target], type: link.type}))
      // console.log(this.internaldata.links)
      this.updateLinks()
    },

    onCategories () {
      const {nodes} = this.internaldata
      console.log('onCategories - internaldata')
      console.log(this.internaldata)
      let categories = {}
      nodes.each(n => {
        (categories[n.parent.data.text] = categories[n.parent.data.text] || []).push(n)
      })
      console.log(categories)
      let categoryArcs = []
      for (const [category, childNodes] of Object.entries(categories)) {
        let start = childNodes[0].x
        console.log(start)
        let end = childNodes[0].x
        let r = childNodes[0].y
        childNodes.forEach(n => {
          start = Math.min(start, n.x)
          end = Math.max(start, n.x)
        })
        let arcLen = childNodes.length
        let strLen = category.length
        let displayName = ''
        if (arcLen * 2 >= strLen) {
          displayName = category
        } else {
          if (arcLen > 1) {
            displayName = category.slice(0, arcLen * 2)
          }
        }
        categoryArcs.push({
          name: category,
          displayName: displayName,
          start: start - this.arcSpacing,
          end: end + this.arcSpacing,
          r: r
        })
      }
      console.log(categoryArcs)

      var palette = ['#D5CFD4', '#EAE4E9', '#FFF1E6', '#FDE2E4', '#FAD2E1', '#E2ECE9',
        '#BEE1E6', '#F0EFEB', '#DFE7FD', '#CDDAFD', '#D2DDFD', '#BEEAD3',
        '#BEEAD3', '#FFE0D6', '#E7CBFF', '#BBE5FF', '#FFFFB0', '#EDEDED']
      categoryArcs.forEach((e, i) => {
        e['color'] = palette[i]
      })
      this.internaldata.categories = categoryArcs
      this.updateCategories()
    },

    instantClean () {
      ['.nodetree', 'text', '.link'].forEach(selector => {
        this.internaldata.g.selectAll(selector).remove()
      })
    },

    redraw () {
      const {root} = this.internaldata
      return root ? Promise.all([this.updateNodes(), this.updateLinks(), this.updateCategories()]) : Promise.resolve('no graph')
    },

    applyTransition (size, {margin}) {
      const {g} = this.internaldata
      const transitiong = g.transition().duration(this.duration)
      this.transformSvg(transitiong, size)
    }
  },

  computed: {
    tree () {
      const size = this.getSize()
      const tree = d3.cluster()
      layout.optimizeSize(tree, size, this.margin, this.textContraint)
      return tree
    },

    margin () {
      return {x: this.marginX, y: this.marginY}
    }
  },

  watch: {
    data (current, old) {
      this.onData(current)
    },

    links (current, old) {
      this.onLinks(current)
    },

    marginX (newMarginX, oldMarginX) {
      this.completeRedraw({margin: {x: oldMarginX, y: this.marginY}})
    },

    marginY (newMarginY, oldMarginY) {
      this.completeRedraw({margin: {x: this.marginX, y: oldMarginY}})
    },

    highlightedNode (newCurrent, oldCurrent) {
      oldCurrent && this.reset(oldCurrent)
      newCurrent && this.showDependencies(newCurrent)
      this.$emit('highlightedNodeChanged', {new: newCurrent, old: oldCurrent})
    }

  }
}
</script>

<style>
.graph .link {
  fill: none;
  stroke: blue;
  stroke-opacity: 0.2;
  stroke-width: 1.5px;
  transition: stroke 0.5s, stroke-opacity 0.5s;
}

.graph.detailed .link.link--source,
.graph.detailed .link.link--target {
  stroke-opacity: 1;
}

.graph.detailed .category.parent-arc,
.graph.detailed .category.parent-arc .categoryText{
  opacity: 1;
}

.graph.detailed .link {
  stroke-opacity: 0.01;
}

.graph .link.link--source {
  stroke: #d62728;
}

.graph .link.link--target {
  stroke: #2ca02c;
}

.graph .nodetree text {
  font: 14px sans-serif;
  transition: opacity 0.5s, fill 0.5s;
  cursor: pointer;
}

.graph .nodetree a:hover{
  text-decoration: none;
}

.graph.detailed .nodetree.node--source text{
  fill: #2ca02c;
}

.graph.detailed .nodetree.node--target text{
  fill: #d62728;
}

.graph.detailed .nodetree.node--selected text,
.graph.detailed .nodetree.node--source text,
.graph.detailed .nodetree.node--target text{
  font-weight: bold;
  opacity: 1;
}

.graph.detailed .nodetree text{
  opacity: 0.2;
}

.graph.detailed .category,
.graph.detailed .categoryText{
  opacity: 0.2;
}

.graph .categoryText:hover{
  cursor: default;
}

path.arc {
  cursor: move;
  fill: #fff;
}

.tooltip {
  position: absolute;
  z-index: 1070;
  display: block;
}
</style>
