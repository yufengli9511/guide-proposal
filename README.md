# Guide
## Naming
- `components` folder
    - Sub folders should be Captilalized camelCase eg. `Cart`
    - Components' filename should be Captilalized camelCase eg. `UserSelector.tsx`
    - Components' filename should be same as Components' name eg.
    `UserSelector.tsx` has `export const UserSelector = async () => {}`

- `utils` folder
    - no need to use export to to `index.ts` just export that function and use named imports
    - any db calls should be put into `services` folder
    - for string/number/format util functions, first check if `lodash` has it
    - Enums should be Uppercase snake_case eg. `export enum COURSE_TYPE {}`
    - Functions/Variables should be camelCase eg. `formatMoney()` | `const currentMarket`

## Auth 
- server components/api route get session, session can be null
`const session = await getServerSession(authOptions)`
- client component, session can be null
` const { data: session } = useSession()`
```typesscript
useEffect(()=>{
    if(session){
        ...
    }
},[session])
```
## Prisma Service
### Example
```typescript
import { prisma } from "@/utils/db"
import { Prisma } from "@prisma/client"

export async function getCampaign(campaignId: number) {
  const campaign = await prisma.campaigns.findFirst({
    where: {
      id: campaignId,
    },
    include: {
      campaign_lists: {
        select: {
          lead_lists: true,
        },
      },
      users_campaigns_sales_user_idTousers: {
        select: {
          id: true,
          first_name: true,
          last_name: true,
        },
      },
    },
  })

  if (!campaign) throw new Error("No campaign found for this id: " + campaignId)

  return campaign
}

export type Campaign = Prisma.PromiseReturnType<typeof getCampaign>
```
- The Error will be catched by nearest error.tsx file
- If you want to only subset of `Campaign`, you can either do `Omit<Campaign,'fieldA'|'fieldB'>` / `Pick<Campaign,'fieldA'|'fieldB'>` / `Partial<Campaign>`
### Raw Query Example
```typescript
import { prisma } from "@/utils/db"
import { Prisma } from "@prisma/client"
// all the types of models are generated
import { email_campaign_senders } from "@prisma/client"

// Pick and add more necessary fields
export type EmailCampaignSender = Pick<
  email_campaign_senders,
  "id" | "from_name" | "from_email" | "company_name" | "address" | "description"
> & { owner_name: string; total_campaigns: number }

export async function getSenders() {
  const senders: EmailCampaignSender[] = await prisma.$queryRaw`
        SELECT ecs.id, ecs.from_name, ecs.from_email, ecs.company_name, ecs.address, ecs.description, CONCAT(u.first_name, ' ', u.last_name) AS owner_name, count(c.id) AS total_campaigns 
        FROM email_campaign_senders AS ecs
            LEFT JOIN campaigns AS c ON c.sender_id = ecs.id
            LEFT JOIN users u ON ecs.owner_user_id = u.id
        GROUP BY ecs.id`

  if (!senders) throw new Error("No senders found")

  return senders
}

export type EmailCampaignSenders = Prisma.PromiseReturnType<typeof getSenders>
```

### Suggestions
- Create individual type for every services
- Define types within the same file as the service function, easier to find
- If there're many parameters of service function, wrap them around an object, for example
`getCampaign(props:{id: number, name: string, is_default: boolean}){...}`
In this way, it's more obvious while getting called
`const campaign = await getCampaign({id, name, is_dafault})`
- Use Prisma queries more to ensure type safety, if you are using raw queries then remember to add types to them.

## API
### Naming
API routes should be `api/course/assign-authors` | `api/cart/create-history`. Use action name as the route name
This will be more clear when view from chrome network tab
### Suggestion
We only use `POST request`, even for get request, we pass all the parameters in `request.body` to ensure type safety
Including passing id in `request.body`
### Example
```typescript
import { prisma } from "@/utils/db"
import { Prisma } from "@prisma/client"
import { NextRequest, NextResponse } from "next/server"

async function getCampaign(where: Prisma.campaignsWhereInput) {
  const campaign = await prisma.campaigns.findFirst({
    where,
    include: {
      campaign_lists: {
        select: {
          lead_lists: true,
        },
      },
      users_campaigns_sales_user_idTousers: {
        select: {
          id: true,
          first_name: true,
          last_name: true,
        },
      },
    },
  })

  if (!campaign) throw new Error("No campaign found for this id: " + where.id)

  return campaign
}

export type Campaign = Prisma.PromiseReturnType<typeof getCampaign>

export interface GetCampaignRequest {
  id: number
  is_default?: boolean
}

export async function POST(request: NextRequest) {
  try {
    const options: GetCampaignRequest = await request.json()

    // You can find `campaignsWhereInput` if you check findFirst/findMany function definition
    const where: Prisma.campaignsWhereInput = {
      id: options.id,
      is_default: options.is_default,
    }

    const campaign = await getCampaign(where)
    return NextResponse.json(campaign)
  } catch (e) {
    return NextResponse.json({ message: (e as Error).message }, { status: 400 })
  }
}

```
- The return type of this api route is `Campaign` and here we don't pass `{success: true}` because if we get the object then it's successful
- We don't throw errors in api route, all errors should return with correct http status code >= 400
- If we are working zod, we can return
    ```typescript
    return NextResponse.json({
              message: "The given data is invalid",
              errors: Object.keys(errors)
    }, { status: 400 })
    ```
- Remember to declare suitable permissions in middleware

## Components
### Suggestion
- separate client/server componenets in each folder
- only create client component when it needs to be interactive i.e. needs `onClick` event or it is in `user-layout`, put it in the lowest level of the tree. Nextjs [Moving Client Components Down the Tree](https://nextjs.org/docs/app/building-your-application/rendering/composition-patterns#moving-client-components-down-the-tree)
- [Working with updating SearchParam query string](https://nextjs.org/docs/app/api-reference/functions/use-search-params#updating-searchparams)
I created a function name `updateQueryString` to adopt this idea















