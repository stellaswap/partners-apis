addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request).catch(error => new Response(error.stack, { status: 400 })))
})

async function handleRequest(request) {

  const returnHeaders = {
    headers: {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET',
      'Access-Control-Allow-Headers': '*',
    },
  }

  const url = new URL(request.url)
  const pathSegments = url.pathname.split('/').filter(segment => segment !== '');
  const pool = pathSegments[0]?.toLowerCase();
  const address = pathSegments[1]?.toLowerCase();
  const subgraphAPI = pathSegments[2]?.toLowerCase();

  if (address == '$address') {
    return new Response(JSON.stringify({
       data: 0,
    }), returnHeaders)
  }

  const subgraphPulsar = `https://gateway-arbitrum.network.thegraph.com/api/${subgraphAPI}/subgraphs/id/G7GRSbr917k92izqozMjNspBqRWp2U91xsj8q6bq8oyH`

  const positionQuery = `
    {
        positions(
          first: 10
          where: {
            pool: "${pool}"
            owner: "${address}"
            transaction_:{
              timestamp_gte: 1726495200
            }
          }
          orderBy: transaction__timestamp
          orderDirection: desc
        ) {
          depositedToken0
          depositedToken1
          pool{
            token0{
              id
            }
            token1{
              id
            }
          }
        }
    }
  `

  const pulsar = await query(subgraphPulsar, positionQuery);

  const positions = pulsar?.data?.positions || [];
  const token0 = positions?.[0]?.pool?.token0?.id
  const token1 = positions?.[0]?.pool?.token1?.id

  const token0PriceQuery = `
    {
      tokenDayDatas(
        first: 1
        orderBy: date
        orderDirection: desc
        where: { token: "${token0}" }
      ) {
        priceUSD
      }
    }
  `

  const token1PriceQuery = `
  {
    tokenDayDatas(
      first: 1
      orderBy: date
      orderDirection: desc
      where: { token: "${token1}" }
    ) {
      priceUSD
    }
  }
`
  const token0QueryResp = await query(subgraphPulsar, token0PriceQuery);
  const token1QueryResp = await query(subgraphPulsar, token1PriceQuery);

  const sums = positions.reduce((acc, position) => {
    acc.depositedToken0 += parseFloat(position.depositedToken0);
    acc.depositedToken1 += parseFloat(position.depositedToken1);
    return acc;
  }, { depositedToken0: 0, depositedToken1: 0 });

  const token0Price = token0QueryResp?.data?.tokenDayDatas?.[0]?.priceUSD ?? 0;
  const token1Price = token1QueryResp?.data?.tokenDayDatas?.[0]?.priceUSD ?? 0;

  return new Response(JSON.stringify({
    wallet: address,
    pool,
    poolCount: positions.length,
    token0: {
      address: token0,
      totalDeposited: sums.depositedToken0,
      price: token0Price,
    },
    token1: {
      address: token1,
      totalDeposited: sums.depositedToken1,
      price: token1Price
    },
    liquidityUSD: sums.depositedToken0 * token0Price + sums.depositedToken1 * token1Price 
  }), returnHeaders)
}

const query = async (api, query) => {
  const resp = await fetch(api, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ query }),
  })
  return await resp.json()
}
