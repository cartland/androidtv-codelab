<h1>Improving animations</h1>

<h2>Goal of checkpoint 5: </h2>

<h2>Concepts</h2>

* <a href="https://developer.android.com/reference/android/support/v17/leanback/app/BackgroundManager.html">BackgroundManager</a>


<h2>Step 1 - Create BackgroundHelper</h2>

<p>Create a class <code>BackgroundHelper<code>.</p>

	public class BackgroundHelper {
		
	    private static long BACKGROUND_UPDATE_DELAY = 200;
		
	    private final Handler mHandler = new Handler();
		
	    private Activity mActivity;
	    private DisplayMetrics mMetrics;
	    private Timer mBackgroundTimer;
	    private String mBackgroundURL;
		
		
	    private Drawable mDefaultBackground;
	    private Target mBackgroundTarget;
		
	    public BackgroundHelper(Activity mActivity) {
	        this.mActivity = mActivity;
	    }
		
	    public void setBackgroundUrl(String backgroundUrl) {
	        this.mBackgroundURL = backgroundUrl;
	    }
	}

<h2>Step 2 - Add an inner class PicassoBackgroundManagerTarget</h2>

    static class PicassoBackgroundManagerTarget implements Target {
        BackgroundManager mBackgroundManager;
		
        public PicassoBackgroundManagerTarget(BackgroundManager backgroundManager) {
            this.mBackgroundManager = backgroundManager;
        }
		
        @Override
        public void onBitmapLoaded(Bitmap bitmap, Picasso.LoadedFrom loadedFrom) {
            this.mBackgroundManager.setBitmap(bitmap);
        }
		
        @Override
        public void onBitmapFailed(Drawable drawable) {
            this.mBackgroundManager.setDrawable(drawable);
        }
		
        @Override
        public void onPrepareLoad(Drawable drawable) {
            // Do nothing, default_background manager has its own transitions
        }
		
        @Override
        public boolean equals(Object o) {
            if (this == o)
                return true;
            if (o == null || getClass() != o.getClass())
                return false;
				
            PicassoBackgroundManagerTarget that = (PicassoBackgroundManagerTarget) o;
			
            if (!mBackgroundManager.equals(that.mBackgroundManager))
                return false;
				
            return true;
        }
		
        @Override
        public int hashCode() {
            return mBackgroundManager.hashCode();
        }
    }

<h2>Step 3 - create a method <code>prepareBackgroundManager</code> to instantiate and prepare the <code>PicassoBackgroundManagerTarget</code></h2>

    public void prepareBackgroundManager() {
        BackgroundManager backgroundManager = BackgroundManager.getInstance(mActivity);
        backgroundManager.attach(mActivity.getWindow());
        mBackgroundTarget = new PicassoBackgroundManagerTarget(backgroundManager);
        mDefaultBackground = mActivity.getResources().getDrawable(R.drawable.default_background);
        mMetrics = new DisplayMetrics();
        mActivity.getWindowManager().getDefaultDisplay().getMetrics(mMetrics);
    }


<h2>Step 5 - add a method <code>updateBackground</code> to load an image</h2>

We are using Picasso tp load and manipulate the image. Once done the instance of the PicassoBackgroundManagerTarget is used to apply the loaded image to the UI.

    protected void updateBackground(String url) {
        Picasso.with(mActivity)
                .load(url)
                .resize(mMetrics.widthPixels, mMetrics.heightPixels)
                .centerCrop()
                .transform(BlurTransform.getInstance(mActivity))
                .error(mDefaultBackground)
                .into(mBackgroundTarget);
        if (null != mBackgroundTimer) {
            mBackgroundTimer.cancel();
        }
    }


<h2>Step 6 - add an inner class <code>UpdateBackgroundTask</code></h2>
<p>This is a subclass of <code>TimerTask</code> to be used to delay updating the background.</p>

    private class UpdateBackgroundTask extends TimerTask {
        @Override
        public void run() {
            mHandler.post(new Runnable() {
                @Override
                public void run() {
                    if (mBackgroundURL != null) {
                        updateBackground(mBackgroundURL);
                    }
                }
            });
        }
    }

<h2>Step 7 - create the methods <code>startBackgroundTimer</code></h2>

<p>In this method the <code>UpdateBackgroundTask</code> is used to schedule a <code>Timer</code> to update of the background.</p>

    public void startBackgroundTimer() {
        if (null != mBackgroundTimer) {
            mBackgroundTimer.cancel();
        }
        mBackgroundTimer = new Timer();
        mBackgroundTimer.schedule(new UpdateBackgroundTask(), BACKGROUND_UPDATE_DELAY);
    }


<h2>Step 8 - Create the class <code>BlurTransform</code></h2>

<p>This is an implementation of the interface <code>com.squareup.picasso</code> which we use to blur the image to be set as background. We start with a auto-generated dummy implementations of the required methods <code>transform</code> and <code>key</code>.</p>

	public class BlurTransform implements Transformation {
	    @Override
	    public Bitmap transform(Bitmap source) {
	        return null;
	    }
		
	    @Override
	    public String key() {
	        return null;
	    }
	}

<h2>Step 9 - Make BlurTransformation a singleton</h2>

We want the BlurTransformation to exists only once and make it a sinlgeton and instantiate a <code>RenderScript</code> in the private constructor which takes a <code>Context</code> as single argument.

	RenderScript rs;
	
    static BlurTransform blurTransform;
	
    protected  BlurTransform() {
        // Exists only to defeat instantiation.
    }
	
    private BlurTransform(Context context) {
        super();
        rs = RenderScript.create(context);
    }
	
    public static BlurTransform getInstance(Context context) {
        if (blurTransform == null) {
            blurTransform = new BlurTransform(context);
        }
        return blurTransform;
    }

<h2>Step 10 - Implement the transform and key method</h2>

The meat of this class is in the <code>transform</code> method which does the trick of blurring the image.

	@Override
    public Bitmap transform(Bitmap bitmap) {
        // Create another bitmap that will hold the results of the filter.
        Bitmap blurredBitmap = Bitmap.createBitmap(bitmap);
		
        // Allocate memory for Renderscript to work with
        Allocation input = Allocation.createFromBitmap(rs, bitmap, Allocation.MipmapControl.MIPMAP_FULL, Allocation.USAGE_SHARED);
        Allocation output = Allocation.createTyped(rs, input.getType());
		
        // Load up an instance of the specific script that we want to use.
        ScriptIntrinsicBlur script = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));
        script.setInput(input);
		
        // Set the blur radius
        script.setRadius(20);
		
        // Start the ScriptIntrinisicBlur
        script.forEach(output);
		
        // Copy the output to the blurred bitmap
        output.copyTo(blurredBitmap);
		
        bitmap.recycle();
		
        return blurredBitmap;
    }
	
    @Override
    public String key() {
        return "blur";
    }

<h2>Step 11 - apply the transformation</h2>

We complete the fluid calls to the Picasso library in the <code>updateBackground</code> method after the <code>centerCrop</code> call. Its pretty simple now.

	.transform(BlurTransform.getInstance(mActivity))


<h2>Step 12 - Add background helper to LeanbackBrowseFragment</h2>

These classes are not used anywhere by now. We add it first to the <code>LeanbackBrowseFragment</code>. Add a member variable <code>bgHelper</code>.

    private BackgroundHelper bgHelper;

and instantiate it at the end of the <code>init</code> method of the <code>LeanbackBrowseFragment</code>:

	bgHelper = new BackgroundHelper(getActivity());
    bgHelper.prepareBackgroundManager();
	

The BackgroundHelper should be updated each time the user changes the selects an item view. So we create a factory method <code>getDefaultSelectedListener</code> which does that:

	protected OnItemViewSelectedListener getDefaultItemSelectedListener() {
        return new OnItemViewSelectedListener() {
            public void onItemSelected(Presenter.ViewHolder itemViewHolder, Object item,
                                       RowPresenter.ViewHolder rowViewHolder, Row row) {
                if (item instanceof Video) {
                    bgHelper.setBackgroundUrl(((Video) item).getThumbUrl());
                    bgHelper.startBackgroundTimer();
                }
				
            }
        };
    }

Now we just register the listener in the <code>init</code> method.

	setOnItemViewSelectedListener(getDefaultItemSelectedListener());

<h2>Step 13 - Add background helper to the <code>VideoDetailsFragment</code></h2>

The background should also be set when the details of a video is shown. So we apply this in the <code>VideoDetailsFragment</code> as well. It simpler here because we don't require a listener. It's just added to the <code>onCrreate</code> method of the fragment.

Add the member variable:

	BackgroundHelper bgHelper;
	
and use it in the <code>onCreate</code> method.

	bgHelper = new BackgroundHelper(getActivity());
	bgHelper.prepareBackgroundManager();
	bgHelper.updateBackground(selectedVideo.getThumbUrl());

<h2>Step 4 - Run app</h2>
