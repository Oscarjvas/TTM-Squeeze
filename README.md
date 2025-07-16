# TTM-Squeeze
#pragma once
#include "IndicatorBase.h"

class TTMSqueezeIndicator : public IndicatorBase
{
private:
    int period = 20;
    double bbMultiplier = 2.0;
    double kcMultiplier = 1.5;
    int momentumLookback = 12;

    Series<double>* sma;
    Series<double>* stddev;
    Series<double>* ema;
    Series<double>* atr;

    Series<bool>* squeezeOn;
    Series<double>* squeezeMomentum;

    PlotSeries* momentumHistogram;
    PlotSeries* squeezeDots;

    double prevSqueezeOn = false;

public:
    TTMSqueezeIndicator()
    {
        AddParameter("BB/KC Period", period);
        AddParameter("BB Multiplier", bbMultiplier);
        AddParameter("KC Multiplier", kcMultiplier);
        AddParameter("Momentum Lookback", momentumLookback);
    }

    void OnInit() override
    {
        sma = SMA(Close, period);
        stddev = StdDev(Close, period);
        ema = EMA(Close, period);
        atr = ATR(period);

        squeezeOn = CreateSeries<bool>("Squeeze On");
        squeezeMomentum = CreateSeries<double>("Squeeze Momentum");

        // Create Histogram Plot for Momentum
        momentumHistogram = AddPlot("Momentum Histogram", PlotType::Histogram, PlotStyle::Solid, Color::Gray);

        // Create Dot Plot for Squeeze Status (Plot above/below candle)
        squeezeDots = AddPlot("Squeeze Dots", PlotType::Points, PlotStyle::Dot, Color::Red);
    }

    void OnBarUpdate() override
    {
        if (CurrentBar < period)
            return;

        double smaVal = sma->GetValue();
        double stddevVal = stddev->GetValue();
        double upperBB = smaVal + bbMultiplier * stddevVal;
        double lowerBB = smaVal - bbMultiplier * stddevVal;

        double emaVal = ema->GetValue();
        double atrVal = atr->GetValue();
        double upperKC = emaVal + kcMultiplier * atrVal;
        double lowerKC = emaVal - kcMultiplier * atrVal;

        bool isSqueezeOn = (lowerBB > lowerKC) && (upperBB < upperKC);
        squeezeOn->SetValue(isSqueezeOn);

        double momentum = Close[0] - Close[momentumLookback];
        squeezeMomentum->SetValue(momentum);

        // Plot Histogram (Green if Positive, Red if Negative)
        if (momentum >= 0)
            momentumHistogram->PlotBar(momentum, Color::Green);
        else
            momentumHistogram->PlotBar(momentum, Color::Red);

        // Plot Dot (Below candle if squeeze on, above if squeeze off)
        if (isSqueezeOn)
            squeezeDots->Plot(Close[0] - (TickSize * 2), Color::Red);  // Dot below candle
        else
            squeezeDots->Plot(Close[0] + (TickSize * 2), Color::Gray); // Dot above candle

        // Alert Logic
        if (CurrentBar > period)
        {
            if (!prevSqueezeOn && isSqueezeOn)
            {
                Alert("Squeeze ON", "Volatility contraction detected");
            }
            else if (prevSqueezeOn && !isSqueezeOn)
            {
                Alert("Squeeze OFF", "Squeeze fired, watch for breakout");
            }
        }

        prevSqueezeOn = isSqueezeOn;
    }
};
