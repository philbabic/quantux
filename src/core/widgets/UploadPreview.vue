
<template>
  <div :class="['MatcWidgetTypeUploadPreview', {'MatcWidgetTypeUploadPreviewImage': hasImage}]" :style="{'backgroundImage': src}"/>
</template>
<script>
import DojoWidget from "dojo/DojoWidget";
import lang from "dojo/_base/lang";
import on from "dojo/on";
import touch from "dojo/touch";
import UIWidget from "core/widgets/UIWidget";

export default {
  name: "UploadPreview",
  mixins: [UIWidget, DojoWidget],
  data: function() {
    return {
      value: "",
      style: {},
      model: {},
      hackValueLabel: false
    };
  },
  computed: {
      hasImage () {
          return this.value && this.value.length > 0
      },
      src () {
          if (this.value) {
                let url = 'url(' + this.value + ')';
                return url
          } else if (this.model) {
                var w = this.model.w * 2;
			    var h = this.model.h * 2;
				var c = document.createElement("canvas");
				var context = c.getContext("2d");
				c.width = w;
				c.height = h;
				h += 0.5;
                w += 0.5;
				var n = 0.5;
				context.moveTo(n, n);
				context.lineTo(w, h);
				context.moveTo(w, n);
				context.lineTo(n, h);
				context.strokeStyle = "#333";
				context.strokeWidth = 2;
				context.imageSmoothingEnabled = false;
				context.stroke();
                let url = 'url(' + c.toDataURL("image/png") + ')';
                return url
          }
          return ''
      }
  },
  methods: {
    postCreate: function() {
      this._borderNodes = [this.domNode];
      this._backgroundNodes = [this.domNode];
      this._shadowNodes = [this.domNode];
      this._paddingNodes = [this.domNode];
      this._labelNodes = [this.domNode];
    },

    wireEvents: function() {
      this.own(this.addClickListener(this.domNode, lang.hitch(this, "onClick")));
      this.own(on(this.domNode, touch.over, lang.hitch(this, "onDomMouseOver")));
      this.own(on(this.domNode, touch.out, lang.hitch(this, "onDomMouseOut")));
    },

    render (model, style, scaleX, scaleY) {
      this.model = model;
      this.style = style;
      this._scaleX = scaleX;
      this._scaleY = scaleY;

      this.setStyle(style, model);
      if (model.props && model.props.label) {
        this.setValue(model.props.label);
      }
    },

    /**
     * Can be overwritten by children to have proper type conversion
     */
    _setDataBindingValue: function(v) {
        if (v.indexOf && v.indexOf('ata:image/png;base64' === 0)) {
            this.setValue(v);
        } else {
            console.error('Only data urls supported', v)
        }
    },

    getValue: function() {
      return this.value;
    },

    setValue: function(value) {
      this.value = value;
    },

    getState: function() {
      return {
        type: "value",
        value: ''
      };
    },

    setState: function(state) {
      if (state && state.type == "value") {
        this.setValue(state.value);
      }
    },

    onClick: function(e) {
      this.stopEvent(e);
      this.emitClick(e);
    }
  },
  mounted() {}
};
</script>