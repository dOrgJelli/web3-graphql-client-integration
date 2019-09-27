# Layer 0
```typescript
const GET_MEMBER_DETAIL = gql`
  query Member($address: String!) {
    member(id: $address) {
      id
      shares
      isActive
      tokenTribute
      delegateKey
    }
  }
`;
```
\+ Easy  
\+ View Specific Optimizations  
\- Not Type Safe === Run-time Errors  
\- No Write Semantics  

# Layer 1
```typescript
interface IMemberState {
  id: string,
  address: Address,
  dao: Address,
  reputation: Address
}

class Member implements IStateful<IMemberState> {

  // Must construct with ID
  // (or equivalent information)
  constructor(id: string) {
    ...
  }

  // IStateful
  public state(): Observable<IMemberState> {
    // returns GraphQL subscription for:
    // member (id: this.id)
  }

  // Collection Queries
  public static search(options: MemberFilter): Observable<Member[]> {
    // returns GraphQL subscription for:
    // members (where: ...options)
  }

  // Entity Linking / Edge Walking
  public dao(): DAO {
    return new DAO(this.state.dao);
  }

  // Write Semantics
  public proposeReward(...) {
    // Proposal Details -> IPFS
    // this.dao().createProposal(..., this.address)
  }
}

// Example usage
const member = new Member("id");

member.state.subscribe(
  (newState) => console.log(newState)
);

Member.search().subscribe(
  (members) => console.log(members)
);

await member.proposeReward(...).send();
```
* Think of the class instance like a minimal property bag, allowing you to interact with its contracts and query additional information from the cache.

\+ Type Safe  
\+ Write Semantics  
\- !Maintenance Nightmare  
\- *Overfetching  

Up to the user to:  
  \- *reduce subscription duplication  
  \- *handle loading  

**\* - Could be solved with clever programming.**  
**\! - Could be solved with code generation.**  

## Usable W/O Subgraph
```typescript
const proposal = new Proposal('0x1234....', arc)
await proposal.vote(...).send()
```

Because the proposal is created with only an `id`, the client will query the subgraph for additional information, such as the address of the contract that the vote needs to be sent to. To make the client usable without having subgraph service available, all Entities have a second way of being created:

```typescript
const proposal = new Proposal({
  id: '0x12455..',
  votingMachine: '0x1111..',
  scheme: '0x12345...'
  }, arc)
```

This will provide the instance with enough information to send transactions without having to query the subgraph for additional information.

# Layer 2
## Option 1
HOC that aggregates subscriptions and passes down the resulting data to the rendering component.

```typescript
export default withSubscription({
  wrappedComponent: DaoMember,
  loadingComponent: <div className={css.loading}>Loading...</div>,
  errorComponent: (props) => <div>{ props.error.message }</div>,
  checkForUpdate: (oldProps, newProps) =>
    oldProps.member.id !== newProps.member.id,
  createObservable: (props: IProps) => props.member.state(),
});
```

## Option 2
HOC + Context Forwarding
```html
<DAO address="0xMy_DAO">

  // Consume Data
  <DAO.Data>
  {(dao: DAOData) => (
    <div>{dao.name}</div>
  )}
  </DAO.Data>

  // Query Collections
  <Members>
    <DAO.Entity>
    <Member.Data>
    {(dao: DAOEntity, member: MemberData) => (
      <>
      <div>{member.name}<div>
      <div>{member.reputation}</div>
      <button onClick={async (e) => {
        await dao.createProposal(..., member.address, ...)
      }}>
      Propose Reward
      </button>
      </>
    )}
    </Member.Data>
    </DAO.Entity>

    // Edge Walking
    <Proposals from="Member as proposer">
      ...
    </Proposals>

  </Members>
</DAOs>
```

https://github.com/mobxjs/mst-gql
