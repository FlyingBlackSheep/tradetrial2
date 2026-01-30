//+------------------------------------------------------------------+
//|                                                  TrendScalp.mq5 |
//|        RSI + ATR + MA Scalping Strategy with Cooldown          |
//+------------------------------------------------------------------+
#property strict

#include <Trade\Trade.mqh>
CTrade trade;

input double lotSize = 0.1;
input int rsiPeriod = 14;
input int atrPeriod = 14;
input double maxSpread = 60; // In points

datetime lastExitTime = 0;
int cooldownBars = 6;  // 6 candles of M5 = 30 minutes

double entryPrice = 0;
double atrAtEntry = 0;
ulong positionTicket = 0;

//+------------------------------------------------------------------+
int OnInit()
{
   return(INIT_SUCCEEDED);
}
//+------------------------------------------------------------------+
void OnTick()
{
   double spread = SymbolInfoInteger(_Symbol, SYMBOL_SPREAD);
   if (spread > maxSpread)
      return;

   // Cooldown logic
   if (lastExitTime > 0)
   {
datetime lastBarTimeArray[1];
if (CopyTime(_Symbol, PERIOD_M5, 1, 1, lastBarTimeArray) <= 0)
    return;

if (TimeCurrent() < (lastExitTime + cooldownBars * 60))  // 6 * 60 = 30 mins
    return; // Still in cooldown, do nothing

   }

   // --- Indicator Handles ---
   int ma100Handle = iMA(_Symbol, PERIOD_H1, 100, 0, MODE_SMA, PRICE_CLOSE);
   int ma200Handle = iMA(_Symbol, PERIOD_H1, 200, 0, MODE_SMA, PRICE_CLOSE);
   int rsiHandle   = iRSI(_Symbol, PERIOD_M5, rsiPeriod, PRICE_CLOSE);
   int atrHandle   = iATR(_Symbol, PERIOD_M5, atrPeriod);

   if (ma100Handle == INVALID_HANDLE || ma200Handle == INVALID_HANDLE || 
       rsiHandle == INVALID_HANDLE || atrHandle == INVALID_HANDLE)
   {
      Print("Indicator handle error.");
      return;
   }

   double ma100[], ma200[], rsi[], atr[];
   if (CopyBuffer(ma100Handle, 0, 0, 1, ma100) <= 0 ||
       CopyBuffer(ma200Handle, 0, 0, 1, ma200) <= 0 ||
       CopyBuffer(rsiHandle, 0, 1, 1, rsi) <= 0 ||
       CopyBuffer(atrHandle, 0, 1, 1, atr) <= 0)
   {
      Print("CopyBuffer error.");
      return;
   }

   double ask = NormalizeDouble(SymbolInfoDouble(_Symbol, SYMBOL_ASK), _Digits);
   double bid = NormalizeDouble(SymbolInfoDouble(_Symbol, SYMBOL_BID), _Digits);
   double currentATR = atr[0];

   // Check if we already have a position
   if (PositionsTotal() == 0)
   {
      // --- Buy Logic ---
      if (ma100[0] > ma200[0] && rsi[0] <= 30)
      {
         entryPrice = ask;
         atrAtEntry = currentATR * 6;
         double stopLoss = NormalizeDouble(entryPrice - atrAtEntry, _Digits);
         if (trade.Buy(lotSize, _Symbol, ask, stopLoss, 0, "RSI Buy"))
         {
            positionTicket = trade.ResultOrder();
         }
         return;
      }

      // --- Sell Logic ---
      if (ma100[0] < ma200[0] && rsi[0] >= 70)
      {
         entryPrice = bid;
         atrAtEntry = currentATR * 6;
         double stopLoss = NormalizeDouble(entryPrice + atrAtEntry, _Digits);
         if (trade.Sell(lotSize, _Symbol, bid, stopLoss, 0, "RSI Sell"))
         {
            positionTicket = trade.ResultOrder();
         }
         return;
      }
   }
   else
   {
      // --- Manage Existing Position ---
      for (int i = 0; i < PositionsTotal(); i++)
      {
         if (PositionGetSymbol(i) == _Symbol)
         {
            double posPrice = PositionGetDouble(POSITION_PRICE_OPEN);
            double posSL = PositionGetDouble(POSITION_SL);
            double posVolume = PositionGetDouble(POSITION_VOLUME);
            long type = PositionGetInteger(POSITION_TYPE);
            double profit = PositionGetDouble(POSITION_PROFIT);
            double currentPrice = (type == POSITION_TYPE_BUY) ? bid : ask;

            double floatingProfit = (type == POSITION_TYPE_BUY) ?
                (currentPrice - entryPrice) : (entryPrice - currentPrice);
            for(int i = PositionsTotal() - 1; i >= 0; i--)
{
      if(PositionGetSymbol(i) == _Symbol)
   {
      ulong ticket = PositionGetInteger(POSITION_TICKET);
      datetime entryTime = (datetime)PositionGetInteger(POSITION_TIME);
      datetime currentTime = TimeCurrent();
      
      if((currentTime - entryTime) >= 10800)  // 3600 seconds = 1 hour
      {
         // Close the position after 1 hour
         if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_BUY)
            trade.PositionClose(ticket);
         else if(PositionGetInteger(POSITION_TYPE) == POSITION_TYPE_SELL)
            trade.PositionClose(ticket);
      }
   }
}

            // --- Move SL to breakeven ---
            if (floatingProfit > atrAtEntry)
            {
               if ((type == POSITION_TYPE_BUY && posSL < entryPrice) ||
                   (type == POSITION_TYPE_SELL && posSL > entryPrice))
               {
                  trade.PositionModify(_Symbol, NormalizeDouble(entryPrice, _Digits), 0);
               }
            }

            // --- Take Profit by RSI ---
            if ((type == POSITION_TYPE_BUY && rsi[0] >= 70) ||
                (type == POSITION_TYPE_SELL && rsi[0] <= 30))
            {
               if (trade.PositionClose(_Symbol))
               {
                  lastExitTime = TimeCurrent();  // Record exit time for cooldown
               }
            }

            // --- Stop Loss hit is handled automatically by broker, we’ll also catch it on next tick ---
         }
      }
   }
}

//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
}
