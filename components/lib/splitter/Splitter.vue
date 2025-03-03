<template>
    <div :class="containerClass">
        <template v-for="(panel, i) of panels" :key="i">
            <component :is="panel" tabindex="-1"></component>
            <div v-if="i !== panels.length - 1" class="p-splitter-gutter" @mousedown="onGutterMouseDown($event, i)" @touchstart="onGutterTouchStart($event, i)" @touchmove="onGutterTouchMove($event, i)" @touchend="onGutterTouchEnd($event, i)">
                <div class="p-splitter-gutter-handle" tabindex="0" :style="gutterStyle" role="separator" :aria-orientation="layout" :aria-valuenow="prevSize" @keyup="onGutterKeyUp" @keydown="onGutterKeyDown($event, i)"></div>
            </div>
        </template>
    </div>
</template>

<script>
import { DomHandler, ObjectUtils } from 'primevue/utils';

export default {
    name: 'Splitter',
    emits: ['resizestart', 'resizeend'],
    props: {
        layout: {
            type: String,
            default: 'horizontal'
        },
        gutterSize: {
            type: Number,
            default: 4
        },
        stateKey: {
            type: String,
            default: null
        },
        stateStorage: {
            type: String,
            default: 'session'
        },
        step: {
            type: Number,
            default: 5
        }
    },
    dragging: false,
    mouseMoveListener: null,
    mouseUpListener: null,
    touchMoveListener: null,
    touchEndListener: null,
    size: null,
    gutterElement: null,
    startPos: null,
    prevPanelElement: null,
    nextPanelElement: null,
    nextPanelSize: null,
    prevPanelSize: null,
    panelSizes: null,
    prevPanelIndex: null,
    timer: null,
    data() {
        return {
            prevSize: null
        };
    },
    mounted() {
        if (this.panels && this.panels.length) {
            let initialized = false;

            if (this.isStateful()) {
                initialized = this.restoreState();
            }

            if (!initialized) {
                let children = [...this.$el.children].filter((child) => DomHandler.hasClass(child, 'p-splitter-panel'));
                let _panelSizes = [];

                this.panels.map((panel, i) => {
                    let panelInitialSize = panel.props && panel.props.size ? panel.props.size : null;
                    let panelSize = panelInitialSize || 100 / this.panels.length;

                    _panelSizes[i] = panelSize;
                    children[i].style.flexBasis = 'calc(' + panelSize + '% - ' + (this.panels.length - 1) * this.gutterSize + 'px)';
                });

                this.panelSizes = _panelSizes;
                this.prevSize = parseFloat(_panelSizes[0]).toFixed(4);
            }
        }
    },
    beforeUnmount() {
        this.clear();
        this.unbindMouseListeners();
    },
    methods: {
        isSplitterPanel(child) {
            return child.type.name === 'SplitterPanel';
        },
        onResizeStart(event, index, isKeyDown) {
            this.gutterElement = event.currentTarget || event.target.parentElement;
            this.size = this.horizontal ? DomHandler.getWidth(this.$el) : DomHandler.getHeight(this.$el);

            if (!isKeyDown) {
                this.dragging = true;
                this.startPos = this.layout === 'horizontal' ? event.pageX || event.changedTouches[0].pageX : event.pageY || event.changedTouches[0].pageY;
            }

            this.prevPanelElement = this.gutterElement.previousElementSibling;
            this.nextPanelElement = this.gutterElement.nextElementSibling;

            if (isKeyDown) {
                this.prevPanelSize = this.horizontal ? DomHandler.getOuterWidth(this.prevPanelElement, true) : DomHandler.getOuterHeight(this.prevPanelElement, true);
                this.nextPanelSize = this.horizontal ? DomHandler.getOuterWidth(this.nextPanelElement, true) : DomHandler.getOuterHeight(this.nextPanelElement, true);
            } else {
                this.prevPanelSize = (100 * (this.horizontal ? DomHandler.getOuterWidth(this.prevPanelElement, true) : DomHandler.getOuterHeight(this.prevPanelElement, true))) / this.size;
                this.nextPanelSize = (100 * (this.horizontal ? DomHandler.getOuterWidth(this.nextPanelElement, true) : DomHandler.getOuterHeight(this.nextPanelElement, true))) / this.size;
            }

            this.prevPanelIndex = index;
            this.$emit('resizestart', { originalEvent: event, sizes: this.panelSizes });
            DomHandler.addClass(this.gutterElement, 'p-splitter-gutter-resizing');
            DomHandler.addClass(this.$el, 'p-splitter-resizing');
        },
        onResize(event, step, isKeyDown) {
            let newPos, newPrevPanelSize, newNextPanelSize;

            if (isKeyDown) {
                if (this.horizontal) {
                    newPrevPanelSize = (100 * (this.prevPanelSize + step)) / this.size;
                    newNextPanelSize = (100 * (this.nextPanelSize - step)) / this.size;
                } else {
                    newPrevPanelSize = (100 * (this.prevPanelSize - step)) / this.size;
                    newNextPanelSize = (100 * (this.nextPanelSize + step)) / this.size;
                }
            } else {
                if (this.horizontal) newPos = (event.pageX * 100) / this.size - (this.startPos * 100) / this.size;
                else newPos = (event.pageY * 100) / this.size - (this.startPos * 100) / this.size;

                newPrevPanelSize = this.prevPanelSize + newPos;
                newNextPanelSize = this.nextPanelSize - newPos;
            }

            this.prevSize = parseFloat(newPrevPanelSize).toFixed(4);

            if (this.validateResize(newPrevPanelSize, newNextPanelSize)) {
                this.prevPanelElement.style.flexBasis = 'calc(' + newPrevPanelSize + '% - ' + (this.panels.length - 1) * this.gutterSize + 'px)';
                this.nextPanelElement.style.flexBasis = 'calc(' + newNextPanelSize + '% - ' + (this.panels.length - 1) * this.gutterSize + 'px)';
                this.panelSizes[this.prevPanelIndex] = newPrevPanelSize;
                this.panelSizes[this.prevPanelIndex + 1] = newNextPanelSize;
            }
        },
        onResizeEnd(event) {
            if (this.isStateful()) {
                this.saveState();
            }

            this.$emit('resizeend', { originalEvent: event, sizes: this.panelSizes });
            DomHandler.removeClass(this.gutterElement, 'p-splitter-gutter-resizing');
            DomHandler.removeClass(this.$el, 'p-splitter-resizing');
            this.clear();
        },
        repeat(event, index, step) {
            this.onResizeStart(event, index, true);
            this.onResize(event, step, true);
        },
        setTimer(event, index, step) {
            this.clearTimer();
            this.timer = setTimeout(() => {
                this.repeat(event, index, step);
            }, 40);
        },
        clearTimer() {
            if (this.timer) {
                clearTimeout(this.timer);
            }
        },
        onGutterKeyUp() {
            this.clearTimer();
            this.onResizeEnd();
        },
        onGutterKeyDown(event, index) {
            switch (event.code) {
                case 'ArrowLeft': {
                    if (this.layout === 'horizontal') {
                        this.setTimer(event, index, this.step * -1);
                    }

                    event.preventDefault();
                    break;
                }

                case 'ArrowRight': {
                    if (this.layout === 'horizontal') {
                        this.setTimer(event, index, this.step);
                    }

                    event.preventDefault();
                    break;
                }

                case 'ArrowDown': {
                    if (this.layout === 'vertical') {
                        this.setTimer(event, index, this.step * -1);
                    }

                    event.preventDefault();
                    break;
                }

                case 'ArrowUp': {
                    if (this.layout === 'vertical') {
                        this.setTimer(event, index, this.step);
                    }

                    event.preventDefault();
                    break;
                }

                default:
                    //no op
                    break;
            }
        },
        onGutterMouseDown(event, index) {
            this.onResizeStart(event, index);
            this.bindMouseListeners();
        },
        onGutterTouchStart(event, index) {
            this.onResizeStart(event, index);
            this.bindTouchListeners();
            event.preventDefault();
        },
        onGutterTouchMove(event) {
            this.onResize(event);
            event.preventDefault();
        },
        onGutterTouchEnd(event) {
            this.onResizeEnd(event);
            this.unbindTouchListeners();
            event.preventDefault();
        },
        bindMouseListeners() {
            if (!this.mouseMoveListener) {
                this.mouseMoveListener = (event) => this.onResize(event);
                document.addEventListener('mousemove', this.mouseMoveListener);
            }

            if (!this.mouseUpListener) {
                this.mouseUpListener = (event) => {
                    this.onResizeEnd(event);
                    this.unbindMouseListeners();
                };

                document.addEventListener('mouseup', this.mouseUpListener);
            }
        },
        bindTouchListeners() {
            if (!this.touchMoveListener) {
                this.touchMoveListener = (event) => this.onResize(event.changedTouches[0]);
                document.addEventListener('touchmove', this.touchMoveListener);
            }

            if (!this.touchEndListener) {
                this.touchEndListener = (event) => {
                    this.resizeEnd(event);
                    this.unbindTouchListeners();
                };

                document.addEventListener('touchend', this.touchEndListener);
            }
        },
        validateResize(newPrevPanelSize, newNextPanelSize) {
            let prevPanelMinSize = ObjectUtils.getVNodeProp(this.panels[0], 'minSize');

            if (this.panels[0].props && prevPanelMinSize && prevPanelMinSize > newPrevPanelSize) {
                return false;
            }

            let newPanelMinSize = ObjectUtils.getVNodeProp(this.panels[1], 'minSize');

            if (this.panels[1].props && newPanelMinSize && newPanelMinSize > newNextPanelSize) {
                return false;
            }

            return true;
        },
        unbindMouseListeners() {
            if (this.mouseMoveListener) {
                document.removeEventListener('mousemove', this.mouseMoveListener);
                this.mouseMoveListener = null;
            }

            if (this.mouseUpListener) {
                document.removeEventListener('mouseup', this.mouseUpListener);
                this.mouseUpListener = null;
            }
        },
        unbindTouchListeners() {
            if (this.touchMoveListener) {
                document.removeEventListener('touchmove', this.touchMoveListener);
                this.touchMoveListener = null;
            }

            if (this.touchEndListener) {
                document.removeEventListener('touchend', this.touchEndListener);
                this.touchEndListener = null;
            }
        },
        clear() {
            this.dragging = false;
            this.size = null;
            this.startPos = null;
            this.prevPanelElement = null;
            this.nextPanelElement = null;
            this.prevPanelSize = null;
            this.nextPanelSize = null;
            this.gutterElement = null;
            this.prevPanelIndex = null;
        },
        isStateful() {
            return this.stateKey != null;
        },
        getStorage() {
            switch (this.stateStorage) {
                case 'local':
                    return window.localStorage;

                case 'session':
                    return window.sessionStorage;

                default:
                    throw new Error(this.stateStorage + ' is not a valid value for the state storage, supported values are "local" and "session".');
            }
        },
        saveState() {
            this.getStorage().setItem(this.stateKey, JSON.stringify(this.panelSizes));
        },
        restoreState() {
            const storage = this.getStorage();
            const stateString = storage.getItem(this.stateKey);

            if (stateString) {
                this.panelSizes = JSON.parse(stateString);
                let children = [...this.$el.children].filter((child) => DomHandler.hasClass(child, 'p-splitter-panel'));

                children.forEach((child, i) => {
                    child.style.flexBasis = 'calc(' + this.panelSizes[i] + '% - ' + (this.panels.length - 1) * this.gutterSize + 'px)';
                });

                return true;
            }

            return false;
        }
    },
    computed: {
        containerClass() {
            return ['p-splitter p-component', 'p-splitter-' + this.layout];
        },
        panels() {
            const panels = [];

            this.$slots.default().forEach((child) => {
                if (this.isSplitterPanel(child)) {
                    panels.push(child);
                } else if (child.children instanceof Array) {
                    child.children.forEach((nestedChild) => {
                        if (this.isSplitterPanel(nestedChild)) {
                            panels.push(nestedChild);
                        }
                    });
                }
            });

            return panels;
        },
        gutterStyle() {
            if (this.horizontal) return { width: this.gutterSize + 'px' };
            else return { height: this.gutterSize + 'px' };
        },
        horizontal() {
            return this.layout === 'horizontal';
        }
    }
};
</script>

<style>
.p-splitter {
    display: flex;
    flex-wrap: nowrap;
}

.p-splitter-vertical {
    flex-direction: column;
}

.p-splitter-panel {
    flex-grow: 1;
}

.p-splitter-panel-nested {
    display: flex;
}

.p-splitter-panel .p-splitter {
    flex-grow: 1;
    border: 0 none;
}

.p-splitter-gutter {
    flex-grow: 0;
    flex-shrink: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: col-resize;
}

.p-splitter-horizontal.p-splitter-resizing {
    cursor: col-resize;
    user-select: none;
}

.p-splitter-horizontal > .p-splitter-gutter > .p-splitter-gutter-handle {
    height: 24px;
    width: 100%;
}

.p-splitter-horizontal > .p-splitter-gutter {
    cursor: col-resize;
}

.p-splitter-vertical.p-splitter-resizing {
    cursor: row-resize;
    user-select: none;
}

.p-splitter-vertical > .p-splitter-gutter {
    cursor: row-resize;
}

.p-splitter-vertical > .p-splitter-gutter > .p-splitter-gutter-handle {
    width: 24px;
    height: 100%;
}
</style>
