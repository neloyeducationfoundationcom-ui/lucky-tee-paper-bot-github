// Lucky Tee - Bitfinex Paper Trading Bot
// STEP 1 VERSION: Market monitoring only.
// This version CANNOT place orders.

const SYMBOL = "tBTCUSD";
const TIMEFRAME = "1m";

const FAST_SMA = 5;
const SLOW_SMA = 20;

async function getTicker() {
  const response = await fetch(
    `https://api-pub.bitfinex.com/v2/ticker/${SYMBOL}`
  );

  if (!response.ok) {
    throw new Error(`Bitfinex ticker error: ${response.status}`);
  }

  const data = await response.json();

  return {
    bid: data[0],
    ask: data[2],
    lastPrice: data[6],
    volume: data[7],
    high: data[8],
    low: data[9],
  };
}

async function getCandles() {
  const url =
    `https://api-pub.bitfinex.com/v2/candles/` +
    `trade:${TIMEFRAME}:${SYMBOL}/hist?limit=30&sort=1`;

  const response = await fetch(url);

  if (!response.ok) {
    throw new Error(`Bitfinex candle error: ${response.status}`);
  }

  return await response.json();
}

function sma(values, length) {
  const selected = values.slice(-length);

  return (
    selected.reduce((total, value) => total + value, 0) /
    selected.length
  );
}

function calculateSignal(candles) {
  // Candle:
  // [MTS, OPEN, CLOSE, HIGH, LOW, VOLUME]

  const closes = candles.map((candle) => Number(candle[2]));

  if (closes.length < SLOW_SMA) {
    return {
      signal: "WAIT",
      fastSMA: null,
      slowSMA: null,
    };
  }

  const fast = sma(closes, FAST_SMA);
  const slow = sma(closes, SLOW_SMA);

  let signal = "HOLD";

  if (fast > slow) {
    signal = "BUY";
  }

  if (fast < slow) {
    signal = "SELL";
  }

  return {
    signal,
    fastSMA: fast,
    slowSMA: slow,
  };
}

export default {
  async fetch(request, env, ctx) {
    try {
      const ticker = await getTicker();
      const candles = await getCandles();

      const strategy = calculateSignal(candles);

      const result = {
        bot: "Lucky Tee Paper Trading Bot",
        status: "MONITORING ONLY",
        tradingEnabled: false,

        market: SYMBOL,
        timeframe: TIMEFRAME,

        price: ticker.lastPrice,
        bid: ticker.bid,
        ask: ticker.ask,

        fastSMA: strategy.fastSMA,
        slowSMA: strategy.slowSMA,

        signal: strategy.signal,

        safety: "NO ORDERS CAN BE PLACED BY THIS VERSION",

        timestamp: new Date().toISOString(),
      };

      return new Response(
        JSON.stringify(result, null, 2),
        {
          headers: {
            "content-type": "application/json;charset=UTF-8",
          },
        }
      );
    } catch (error) {
      return new Response(
        JSON.stringify(
          {
            bot: "Lucky Tee Paper Trading Bot",
            status: "ERROR",
            error: error.message,
          },
          null,
          2
        ),
        {
          status: 500,
          headers: {
            "content-type": "application/json;charset=UTF-8",
          },
        }
      );
    }
  },
};