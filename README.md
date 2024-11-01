# PanZoomRotate
ItemTouchListener for the view to pan, zoom, and rotate with 1 or 2+ fingers

## Implementation
Just attach an instance of PanZoomRotateListener to your view:

```kotlin
binding.imageView.setOnTouchListener(PanZoomRotateListener())
```

## Code

```kotlin
class PanZoomRotateListener : View.OnTouchListener {
    private var startPointerPosition: PointerPositions = PointerPositions.None
    private var baseTranslationX = 0f
    private var baseTranslationY = 0f
    private var baseAngle: Float = 0f
    private var baseScale: Float = 1f

    override fun onTouch(view: View?, event: MotionEvent?): Boolean {
        if (view == null || event == null) return false
        when (event.actionMasked) {
            MotionEvent.ACTION_POINTER_DOWN,
            MotionEvent.ACTION_POINTER_UP,
                -> {
                startPointerPosition = event.toPointerPositions(view)
                updateBaseTransformations(view)
            }

            MotionEvent.ACTION_CANCEL -> {
                startPointerPosition = PointerPositions.None
                updateBaseTransformations(view)
            }

            MotionEvent.ACTION_DOWN -> {
                startPointerPosition = event.toPointerPositions(view)
                updateBaseTransformations(view)
            }

            MotionEvent.ACTION_UP -> {
                startPointerPosition = PointerPositions.None
                updateBaseTransformations(view)
                view.performClick()
            }

            MotionEvent.ACTION_MOVE -> {
                val currentPointerPositions = event.toPointerPositions(view)
                startPointerPosition.panning(currentPointerPositions)?.let { (x, y) ->
                    view.translationX = baseTranslationX + x
                    view.translationY = baseTranslationY + y
                }
                startPointerPosition.zooming(currentPointerPositions)?.let { scaleFactor ->
                    view.scaleX = baseScale * scaleFactor
                    view.scaleY = baseScale * scaleFactor
                }
                startPointerPosition.rotation(currentPointerPositions)?.let { angle ->
                    view.rotation = baseAngle + angle
                }
            }
        }
        return true
    }

    private fun updateBaseTransformations(view: View) {
        baseTranslationX = view.translationX
        baseTranslationY = view.translationY
        baseAngle = view.rotation
        baseScale = view.scaleX
    }
}

private sealed interface PointerPositions {
    object None : PointerPositions
    class Single(motionEvent: MotionEvent) : PointerPositions {
        val x: Float = motionEvent.rawX
        val y: Float = motionEvent.rawY
    }

    class Double(motionEvent: MotionEvent, view: View) : PointerPositions {
        val x1: Float = motionEvent.rawX
        val y1: Float = motionEvent.rawY
        val x2: Float
        val y2: Float

        init {
            val lastPointerIndex = motionEvent.pointerCount - 1
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
                x2 = motionEvent.getRawX(lastPointerIndex)
                y2 = motionEvent.getRawY(lastPointerIndex)
            } else {
                val rawCoordinatesOfLastPointer =
                    motionEvent.getRawCoordinates(lastPointerIndex, view)
                x2 = rawCoordinatesOfLastPointer.first
                y2 = rawCoordinatesOfLastPointer.second
            }
        }

        val middleX = (x1 + x2) / 2
        val middleY = (y1 + y2) / 2

        val distance: Float
            get() = sqrt((x2 - x1).pow(2) + (y2 - y1).pow(2))

        val angle: Float
            get() = atan2(y2 - y1, x2 - x1) * RADIANS_TO_DEGREES

        private fun MotionEvent.getRawCoordinates(pointerIndex: Int, view: View): Pair<Float, Float> {
            val screenMatrix = Matrix()
            screenMatrix.postRotate(view.rotation, view.pivotX, view.pivotY)
            screenMatrix.postScale(view.scaleX, view.scaleY, view.pivotX, view.pivotY)
            screenMatrix.postTranslate(view.left.toFloat(), view.top.toFloat())
            val viewToScreenCoords = floatArrayOf(getX(pointerIndex), getY(pointerIndex))
            screenMatrix.mapPoints(viewToScreenCoords)
            return viewToScreenCoords[0] to viewToScreenCoords[1]
        }
    }

    companion object {
        private const val RADIANS_TO_DEGREES = 57.29577f
        fun MotionEvent.toPointerPositions(view: View): PointerPositions = when (pointerCount) {
            0 -> None
            1 -> Single(this)
            else -> Double(this, view)
        }

        /**
         * 2+ pointers panning is disabled for devices with Android below 10
         */
        fun PointerPositions.panning(current: PointerPositions): Pair<Float, Float>? = when {
            this is Single && current is Single -> {
                (current.x - x) to (current.y - y)
            }

            Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q
                    && this is Double && current is Double -> {
                (current.middleX - middleX) to (current.middleY - middleY)
            }

            else -> null
        }

        fun PointerPositions.zooming(current: PointerPositions): Float? {
            if (this !is Double || current !is Double) return null

            return current.distance / distance
        }

        fun PointerPositions.rotation(current: PointerPositions): Float? {
            if (this !is Double || current !is Double) return null

            return current.angle - angle
        }
    }
}
```
