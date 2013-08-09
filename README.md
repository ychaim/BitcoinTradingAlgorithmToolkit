BitcoinTradingAlgorithmToolkit
==============================

A framework for logging, simulating, and analyzing prices of crypto currencies on various exchanges using technical analysis, fuzzy logic, and neural networks.

If you like this ... BTC: 1GqES1gCLdHd2gE2P6L1gqyYjf87bjn8mA

## What is this?

Can machine learning be used to trade Bitcoin profitably? Can ML algorithms predict the prices based on technical indicators? This was an experiment in seeing how well a neural network or fuzzy logic could learn the "rules" of the various Bitcoin markets' price movements. (If they exist at all.) To do that I needed a framework for logging and then replaying price movements and a way to show them to the algorithms. I dug up some of the most common and more obscure indicators referenced in some classic NN-trading literature from old books, mathematical formulas, and other languages' implementations.

Careful using this code. Especially if you're betting actual money. (Insane!) I tested this code a lot, but it's a hack-y mess and it could easily still have a good amount of bugs. (Especially when it comes to oddities with numpy's NaN and the fact that barely any of the indicators accounted for division by zero.) I didn't go into this with the idea that anyone else would ever use it.

DISCLAIMER: Remember that trading is gambling and that technical indicators are just a way to fool us into thinking we can see the future.

### Logging Exchange Data

`logtick.py` is what I used to log ticker, market depth, and volume information from BTC-e and Mt. Gox to a CSV file. There are better logging tools and scripts out there, but you can find them yourself. Mine uses multiprocessing and compresses the entire depth information for that moment. If you want to train a machine learning algorithm, especially a neural network, you need tons of data. The more you get, the better at generalizing your NN can be. I set it to log day and night for about a month. Since my intent was to trade LTC automatically, I was interested in LTC volume, not BTC-e BTC or GOX volume, so that was all I logged.

`logtick.py` logs the following fields to CSV:

    gox_last, gox_buysell, gox_time,
    btc_usd_last, btc_usd_buysell, btc_usd_time,
    ltc_btc_last, ltc_btc_buysell, ltc_btc_time,
    ltc_last, ltc_buysell, ltc_time,
    ltc_depth, ltc_depth_time, ltc_24h_volume

### Analyzing and Simulating

`data.py` has most of the code used in processing the logged price data, playing it back to us (used in simulating what would happen if we traded using X-rules or Y-strategy), and statistically analyzing the logged data using technical indicators, averages, standard deviations, etc.

The two classes in `data.py` (`Data` and `Coin`) make up the core of the Bitcoin analysis functionality. The Coin class holds and generates ticker prices and averages, std devs, and technical indicators based on them. The Data class wraps the Coin class(es) and loads ticker prices into them (either from the actual exchange over the web, or from log files from disk). It allows for mutiple exchanges to be loaded and ultimately compared (i.e., GOX Bitcoin and BTC-e Litecoin).

The usage of it evolved over time and can be used several ways. Lets say that we want to load some logged Litecoin price data from disk and then run some rolling means, all available technical indicators, compound returns, and some standard deviations on it. We also want to look at the data in 10-minute OHLC (open, high, low, close) chunks:

    import pandas as pd
    import data

    # 10-minute timeframe
    time_str = "10min"

    # our litecoin, statistical analysis options
    ltc_opts = \
    {  "debug": False,
       "relative": False,
       "calc_rolling": True,
       "rolling": { time_str : {  12: pd.DataFrame(),
                                  24: pd.DataFrame(),
                                  50: pd.DataFrame() } },
       "calc_mid": True,
       "calc_ohlc": True,
       "ohlc": { time_str : pd.DataFrame()  },
       "calc_indicators": True,
       "indicators":{ "RSI"  : { "data": pd.DataFrame(), "n":14 },
                      "ROC"  : { "data": pd.DataFrame(), "n":20 },
                      "AMA"  : { "data": pd.DataFrame(), "n":10, "fn":2.5, "sn":30 },
                      "CCI"  : { "data": pd.DataFrame(), "n":20 },
                      "FRAMA": { "data": pd.DataFrame(), "n":10 },
                      "RVI2" : { "data": pd.DataFrame(), "n":14, "s":10 },
                      "MACD" : { "data": pd.DataFrame(), "f":12, "s":26, "m":9 },
                      "ADX"  : { "data": pd.DataFrame(), "n":14 },
                      "ELI"  : { "data": pd.DataFrame(), "n":14 },
                      "TMI"  : { "data": pd.DataFrame(), "nb":10, "nf":5 }
                    },
       "calc_std": True,
       "std": { 13 : pd.DataFrame(), 21 : pd.DataFrame(), 34 : pd.DataFrame() },
       "calc_crt": True,
       "crt": { 1: pd.DataFrame(),
                2: pd.DataFrame(),
                3: pd.DataFrame(),
                5: pd.DataFrame(),
                8: pd.DataFrame() },
       "instant": True,
       "time_str": time_str }

    # warp and instant = True forces Data to calculate everything all in one
    # pass. Other options would make it run in simulated real time. Load 
    # filename called "test.csv" from ./logs/ ... Data
    # assumes all logs are in the local directory called logs
    d = data.Data( warp=True, instant=True, ltc_opts=ltc_opts,
                   time_str=time_str, filename="test.csv")

    # Pull out all indicators into a single pandas dataframe. Prefix all rows
    # with "LTC"
    ltc = d.ltc.combine( "LTC")

From there, `ltc` would be a dataframe filled with all the values we'd need to start training our neural network or use with any other algorithm.

### Technical indicators

I tried a lot of different inputs to the machine learning algorithms. Technical indicators seemed to be the most common strategy in the literature. `indicators.py` implements the following technical indicators and analysis functions:

- AVERAGE DIRECTIONAL MOVEMENT INDEX (ADI)
- ADAPTIVE MOVING AVERAGE (AMA)
- COMMODITY CHANNEL INDEX (CCI)
- COMPOUND RETURN
- EHLER'S LEADING INDICATOR (ELI)
- FRACTAL ADAPTIVE MOVING AVERAGE (FRAMA)
- MACD
- NORMALIZATION
- RATE OF CHANGE (ROC)
- RELATIVE STRENGTH INDEX (RSI)
- RELATIVE VOLUME INDEX (w/ Inertia) (RVI)
- STANDARD DEVIATIONS
- TREND MOVEMENT INDEX (TMI)

I chose these specific indicators because they were featured in a lot of "classic" algorithmic trading articles, particularly "Forecasting Foreign Exchange Rates Using Recurrent Neural Networks" by Paolo Tenti.

### Converting the Data for Training

`dtools.py` has a lot of convenience functions for turning our dataframes into input and target matrices. The `gen_ds` function in particular will perform this transformation (continuing from the above example):

    import dtools

    # take our ltc dataframe, and get targets (prices in next 10 minutes)
    # in the form of compound return prices (other options are "PRICES", 
    # which are raw price movements)
    dataset, tgt = dtools.gen_ds( ltc, 1, ltc_opts, "CRT")

That should have stripped any starting Nans (some of the technical indicators need time to "warm up") and shifted our prices forward into `tgt`. We will use `dataset` as our inputs and `tgt` as the training examples.

### Using with a Neural Network

There are a couple python Neural Network packages. Only two had even close to enough functionality to use: [pybrain](http://pybrain.org/) and [neurolab](https://code.google.com/p/neurolab/). I also experimented with using [Python-Matlab-Wormholes](http://code.google.com/p/python-matlab-wormholes/) and using the Matlab Neural Network Toolkit (which has some great implementations of recurrent/time-delayed neural networks). Here's an example with pybrain:

    from pybrain.datasets            import SupervisedDataSet
    from pybrain.tools.shortcuts     import buildNetwork
    from pybrain.supervised.trainers import RPropMinusTrainer
    from pybrain.structure           import RecurrentNetwork, FullConnection
    from pybrain.structure.modules   import LinearLayer, TanhLayer
    import numpy as np
    from matplotlib import pylab as plt

    # initialize a pybrain dataset
    DS = SupervisedDataSet( len(dataset.values[0]), np.size(tgt.values[0]) )

    # fill it
    for i in xrange( len( dataset)):
      DS.appendLinked( dataset.values[i], [ tgt.values[i]] )

    # split 70% for training, 30% for testing
    train_set, test_set = DS.splitWithProportion( .7)

    # build our recurrent network with 10 hidden neurodes, one recurrent
    # connection, using tanh activation functions
    net = RecurrentNetwork()
    hidden_neurodes = 10
    net.addInputModule( LinearLayer(len( train_set["input"][0]), name="in"))
    net.addModule( TanhLayer( hidden_neurodes, name="hidden1"))
    net.addOutputModule( LinearLayer( len( train_set["target"][0]), name="out"))
    net.addConnection(FullConnection(net["in"], net["hidden1"], name="c1"))
    net.addConnection(FullConnection(net["hidden1"], net["out"], name="c2"))
    net.addRecurrentConnection(FullConnection(net["out"], net["hidden1"], name="cout"))
    net.sortModules()
    net.randomize()

    # train for 30 epochs using the rprop- training algorithm
    trainer = RPropMinusTrainer( net, dataset=train_set, verbose=True )
    trainer.trainOnDataset( train_set, 30)

    # test on training set
    predictions_train = np.array( [ net.activate( train_set["input"][i])[0] for i in xrange( len(train_set)) ])
    plt.plot( train_set["target"], c="k"); plt.plot( predictions_train, c="r"); plt.show()

    # and on test set
    predictions_test = np.array( [ net.activate( test_set["input"][i])[0] for i in xrange( len(test_set)) ])
    plt.plot( test_set["target"], c="k"); plt.plot( predictions_test, c="r"); plt.show()

And there you have it. If you used test.csv, you probably didn't get very much data or a very accurate model. But it's a start.

### And beyond ...

If you did all this using the test.csv dataset, only about a day's worth of price data, your model isn't going to be very good, in general. Also, the simple recurrent neural network with generic mean-squared error function turned out, in my opinion, to not be a very good way to represent "error" in predicting price changes. When all you're trying to do it get as "close as possible" to the price change value, the NN seemed to hover around zero, without regard to positive or minus. There are alternative error functions that could work better for our ultimate goal: will the price go up or down? One idea I read about was to use an error function that penalizes more heavily if our model predicts the wrong direction of price movement.

Trying to hack this into pybrain turned out to be difficult because the error function seems to be hardcoded into the NN code. Neurolab makes it easier to implement custom error functions and I had some success using a "weighted directional symmetry" (WDS) error function. But it wasn't super-accurate, either, and didn't generalize well far into the future.

There was also the problem of optimizing the parameters to each technical indicator. I approached this problem by using genetic algorithms to find the most accurate combinations of which technical indicators and statistical analysis methods to use and which parameters to use with them. I used [DEAP](https://code.google.com/p/deap/) for this and was happy with the project and result. The end result still wasn't anywhere near perfect at predicting the price movements, though, so I tried another strategy ...

Turning to fuzzy-logic, I had a little more success (theoretically profitable models) using [pyfuzzy](http://pyfuzzy.sourceforge.net/) (which took a lot of hacking to import for some reason), a very minimalistic number of indicators, and DEAP-based genetic algorithms for technical indicator parameter optimization. These models didn't stay profitable for very long at all, though.

## Dependencies

- pandas
- numpy
- matplotlib (for plots)
- eventlet (for ticker logging)