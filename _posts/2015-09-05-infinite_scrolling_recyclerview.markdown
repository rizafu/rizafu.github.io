---
layout: post
title: Infinite Scrolling with RecyclerView
date: '2015-09-05'
cover_image: '/content/images/2015/9/infinite-scrolling.png'
tags:
- RecyclerView
- android
- infinite-scrolling
- footer
- ProgressBar
- adapter
summary: An example of how-to implement an infinite scrolling adapter for a RecyclerView, with a ProgressBar footer.
---

## Requirements
1. Infinite scrolling dataSet (Duh!)
2. Inform user of process of fetching new data without blocking main view

## Solution
As usual, Gist can be found [here](https://gist.github.com/mSobhy90/cf7fa98803a0d7716a4a)

### The solution consists of 3 main parts:
1. [AbstractRecyclerViewFooterAdapter](https://gist.github.com/mSobhy90/cf7fa98803a0d7716a4a#file-abstractrecyclerviewfooteradapter-java): a data-independent abstract Adapter that can be re-used to achieve same functionality (infinite scrolling with footer ProgressBar)
2. [RecyclerViewFooterAdapterImpl](https://gist.github.com/mSobhy90/cf7fa98803a0d7716a4a#file-recyclerviewfooteradapterimpl-java): an example Adapter that implements AbstractRecyclerViewFooterAdapter to facilitate the understanding of the main points of the Adapter
3. [MyActivity](https://gist.github.com/mSobhy90/cf7fa98803a0d7716a4a#file-myactivity-java): the glue of them all.

Let's resolve the pieces, one by one:

#### 1. AbstractRecyclerViewFooterAdapter

Here is where all the magic of infinite scrolling happens (except loading more data)

##### Getting scroll events using ```recyclerView.addOnScrollListener(RecyclerView.OnScrollListener);```

{% highlight java %}

if (recyclerView.getLayoutManager() instanceof LinearLayoutManager) {
    final LinearLayoutManager linearLayoutManager = (LinearLayoutManager) recyclerView.getLayoutManager();
    recyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
        @Override
        public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
            totalItemCount = linearLayoutManager.getItemCount();
            visibleItemCount = linearLayoutManager.getChildCount();
            firstVisibleItem = linearLayoutManager.findFirstVisibleItemPosition();

            if (loading) {
                if (totalItemCount > previousTotal) {
                    loading = false;
                    previousTotal = totalItemCount;
                }
            }
            if (!loading && (totalItemCount - visibleItemCount)
                    <= (firstVisibleItem + VISIBLE_THRESHOLD)) {
                // End has been reached

                addItem(null);
                if (onLoadMoreListener != null) {
                    onLoadMoreListener.onLoadMore();
                }
                loading = true;
            }
        }
    });
}

{% endhighlight %}

###### Focus points:
1. ```totalItemCount```: The total number of items in the RecyclerView (Visible + Invisible)
2. ```visibleItemCount```: The number of items visible to the user right now
3. ```firstVisibleItem```: The index of the first visible item
4. ```loading```: A ```boolean``` to differentiate between if more items are being loaded or not
5. ```VISIBLE_THRESHOLD```: a ```final int``` variable (in this example <b>equals 5</b>) which specifies the number of remaining elements before starting to load next batch
6. ```onLoadMoreListener```: a listener that propagates back to the ```Activity``` the task of loading more items
7. ```addItem(null)```: We here tail our adapter dataset with a pre-definied value ```"null"``` that the adapter is gonna check on to inflate the footer view (check following code snippet)

##### Inflating the right ```RecyclerView.ViewHolder``` based on ```position```

{% highlight java %}

@Override
public int getItemViewType(int position) {
    return dataSet.get(position) != null ? ITEM_VIEW_TYPE_BASIC : ITEM_VIEW_TYPE_FOOTER;
}

@Override
public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
    if (viewType == ITEM_VIEW_TYPE_BASIC) {
        return onCreateBasicItemViewHolder(parent, viewType);
    } else if (viewType == ITEM_VIEW_TYPE_FOOTER) {
        return onCreateFooterViewHolder(parent, viewType);
    } else {
        throw new IllegalStateException("Invalid type, this type ot items " + viewType + " can't be handled");
    }
}

@Override
public void onBindViewHolder(RecyclerView.ViewHolder genericHolder, int position) {
    if (getItemViewType(position) == ITEM_VIEW_TYPE_BASIC) {
        onBindBasicItemView(genericHolder, position);
    } else {
        onBindFooterView(genericHolder, position);
    }
}

{% endhighlight %}

###### Focus points:
1. ```dataSet.get(position) != null```: A check for the pre-definied value that will indicate footer
2. ```onCreateViewHolder(ViewGroup, int)```: calling ```abstract``` methods that will create the view for each corresponding ```viewType```
3. ```onBindViewHolder(RecyclerView.ViewHolder, int)```: calling ```abstract``` methods that will do the binding of the created ```RecyclerView.ViewHolder``` to their actual values

##### Defining the ```abstract``` methods we just called

{% highlight java %}

public abstract RecyclerView.ViewHolder onCreateBasicItemViewHolder(ViewGroup parent, int viewType);

public abstract void onBindBasicItemView(RecyclerView.ViewHolder genericHolder, int position);

public RecyclerView.ViewHolder onCreateFooterViewHolder(ViewGroup parent, int viewType) {
    //noinspection ConstantConditions
    View v = LayoutInflater.from(parent.getContext())
            .inflate(R.layout.progress_bar, parent, false);
    return new ProgressViewHolder(v);
}

public void onBindFooterView(RecyclerView.ViewHolder genericHolder, int position) {
    ((ProgressViewHolder) genericHolder).progressBar.setIndeterminate(true);
}

public static class ProgressViewHolder extends RecyclerView.ViewHolder {
    @InjectView(R.id.progressBar)
    public ProgressBar progressBar;

    public ProgressViewHolder(View v) {
        super(v);
        ButterKnife.inject(this, v);
    }
}

{% endhighlight %}

###### Focus points:
1. ```onCreateFooterViewHolder(ViewGroup, int)```: I opted to implementing this by default cause I didn't have reason for a different type of footer, and you can easily ```@Override``` this method to create your own view (or to customize ```ProgressBar```!)
2. ```((ProgressViewHolder) genericHolder).progressBar.setIndeterminate(true);```: Same point as ```onCreateFooterViewHolder``` and must be overridden as well if you're overriding ```onCreateFooterViewHolder```

##### Defining the ```resetItems``` functionality

{% highlight java %}

public void resetItems(@NonNull List<T> newDataSet) {
    loading = true;
    firstVisibleItem = 0;
    visibleItemCount = 0;
    totalItemCount = 0;
    previousTotal = 0;

    dataSet.clear();
    addItems(newDataSet);
}

public void addItems(@NonNull List<T> newDataSetItems) {
    dataSet.addAll(newDataSetItems);
    notifyDataSetChanged();
}

public void addItem(T item) {
    if (!dataSet.contains(item)) {
        dataSet.add(item);
        notifyItemInserted(dataSet.size() - 1);
    }
}

public void removeItem(T item) {
    int indexOfItem = dataSet.indexOf(item);
    if (indexOfItem != -1) {
        this.dataSet.remove(indexOfItem);
        notifyItemRemoved(indexOfItem);
    }
}

{% endhighlight %}

###### Focus points:
* None, I believe everything is self-explanatory.

That was the end of ```AbstractRecyclerViewFooterAdapter```, full class can be found [here](https://gist.github.com/mSobhy90/cf7fa98803a0d7716a4a#file-abstractrecyclerviewfooteradapter-java).

#### 2. RecyclerViewFooterAdapterImpl

Here is an example of how I extended ```AbstractRecyclerViewFooterAdapter``` to fit my needs

##### Class creation

{% highlight java %}

public class RecyclerViewFooterAdapterImpl extends AbstractRecyclerViewFooterAdapter<MyModelType> {

    private Activity activity;
    private SortType currentSortType;

    public RecyclerViewFooterAdapterImpl(RecyclerView recyclerView, List<MyModelType> dataset, OnLoadMoreListener onLoadMoreListener, Activity activity, SortType currentSortType) {
        super(recyclerView, usedVehicleEngines, onLoadMoreListener);
        this.activity = activity;
        this.currentSortType = currentSortType;
    }

    // Rest coming later on...
}

{% endhighlight %}

###### Focus points:
1. ```RecyclerViewFooterAdapterImpl extends AbstractRecyclerViewFooterAdapter<MyModelType>```: You can use whatever data type you need, we're ```<T>```ing the hell out of ```AbstractRecyclerViewFooterAdapter```
2. Extra info can be added (and encouraged) for usage inside the ```RecyclerViewFooterAdapterImpl``` instance, for ex: ```private SortType currentSortType;```

##### Implementing ```abstract``` functions

{% highlight java %}

@Override
public RecyclerView.ViewHolder onCreateBasicItemViewHolder(ViewGroup parent, int viewType) {
    View v = LayoutInflater.from(parent.getContext())
            .inflate(R.layout.my_normal_item_custom_layout, parent, false);
    return new MyOwnHolder(v);
}

@Override
public void onBindBasicItemView(RecyclerView.ViewHolder genericHolder, int position) {
    final MyOwnHolder holder = (MyOwnHolder) genericHolder;
    final MyModelType currentlySelectedItem = getItem(position);
    // DO YOU BINDING MAGIC HERE
}

{% endhighlight %}

###### Focus points:
1. In ```onCreateBasicItemViewHolder```: I'm inflating the XML that contains my normal item layout, and returning the ```View``` into ```MyOwnHolder``` to be used in ```onBindBasicItemView```
2. In ```onBindBasicItemView```: I'm getting ```MyOwnHolder``` and the currently selected item ```getItem(postition)``` and then populating the data the way I wish

##### Miscellaneous part

{% highlight java %}

// A wrapper version of resetItems to allow for manipulating Impl adapter data before reseting dataSet
public void resetItems(@NonNull List<MyModelType> newDataSet, SortType sortType) {
    currentSortType = sortType;
    resetItems(newDataSet);
}

{% endhighlight %}

###### Focus point:
* I'm wrapping ```AbstractRecyclerViewFooterAdapter.resetItems()``` into my own to maintain correct data state consistency of my own instance.

That was the end of ```RecyclerViewFooterAdapterImpl```, full class can be found [here](https://gist.github.com/mSobhy90/cf7fa98803a0d7716a4a#file-recyclerviewfooteradapterimpl-java).

#### 3. MyActivity

Here is an example Activity ```MyActivity``` that uses ```RecyclerViewFooterAdapterImpl``` (<b>The glue of everything</b>)

##### Class creation

{% highlight java %}

public class MyActivity extends ActionBarActivity {

    @InjectView(R.id.myRecyclerView)
    RecyclerView mRecyclerView;

    RecyclerViewFooterAdapterImpl mAdapter;

    //Rest coming later on...
}

{% endhighlight %}

###### Focus points:
* Just my basic ```Activity``` with an instance of ```RecyclerView``` & ```RecyclerViewFooterAdapterImpl```

##### Working with ```RecyclerViewFooterAdapterImpl``` and implementing ```OnLoadMoreListener```

{% highlight java %}

@Override
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  setContentView(getLayoutResource());

  RecyclerView.LayoutManager mLayoutManager = new LinearLayoutManager(this);
  mRecyclerView.setLayoutManager(mLayoutManager);

  mAdapter = new RecyclerViewFooterAdapterImpl(mRecyclerView, vehicleModelEngines, new OnLoadMoreListener() {
        @Override
        public void onLoadMore() {
            callEndpointForPage(vehicleMakeModel.getId(), new Runnable() {
                @Override
                public void run() {
                    mAdapter.removeItem(null); // don't forget to remove the progress bar representative value
                    updateAdapter(false);
                }
            });
        }
    }, this, currentSortType);
  mRecyclerView.setAdapter(mAdapter);
}

private void updateAdapter(boolean shouldGoUp) {
    List<MyModelType> myDataSet = new ArrayList<>(MyModelType);
    mAdapter.resetItems(myDataSet, currentSortType);
    if (shouldGoUp) {
        if (mAdapter.getFirstVisibleItem() <= 50) {
            mRecyclerView.smoothScrollToPosition(0);
        } else {
            mRecyclerView.scrollToPosition(0);
        }
    }
}

{% endhighlight %}

###### Focus points:
1. ```mRecyclerView.setLayoutManager(mLayoutManager);```: setting layout manager to a ```LinearLayoutManager``` to be able to get firstVisibleItem
2. ```onLoadMore```: call your own endpoint that's gonna fetch next batch of data from your back-end
3. ```mAdapter.removeItem(null);```: remove the pre-definied value of progress/footer from the dataset to stop ```ProgressBar``` from appearing
4. ```updateAdapter(boolean);```: where I get an updated dataSet from the endpoint, and reset items with the whole new set of data.
5. ```shouldGoUp```: just a ```boolean``` to determine if user should be sent to the top of the dataset (ex: when sort is changed).
6. ```if (mAdapter.getFirstVisibleItem() <= 50)```: a check to see if we can have a ```smoothScrollToPosition``` that's not gonna take too much time or not

That was the end of ```MyActivity```, full class can be found [here](https://gist.github.com/mSobhy90/cf7fa98803a0d7716a4a#file-myactivity-java).

## Final words:
Here are the results of this implementation:

[![Sample video](/content/images/2015/9/infinite-scrolling-screenshot.png)](http://youtu.be/v04i0R_4aR0 "Infinite Scrolling with RecyclerView - Click to Watch!")

Hope you find this info helpful and get to implement an awesome infinite-scrolling activity.

---

Bits and pieces:
-  [StackOverflow question/answer](http://stackoverflow.com/a/30691092/1328476) by [Vilen](http://stackoverflow.com/users/855843/vilen-melkumyan)
- Github [gist](https://gist.github.com/sebnapi/fde648c17616d9d3bcde) by [Sebnapi](https://gist.github.com/sebnapi)

---
Cover photo: [Infinite Scrolling](http://designshack.net/wp-content/uploads/infinite-scrolling-featured.jpg)
