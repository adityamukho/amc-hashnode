## A Closer Look at Delta Arithmetic

In their [1996 paper published in ACM Transactions on Database Systems](https://www.seas.upenn.edu/~zives/03s/cis650/P370.PDF), Ghandeharizadeh et al. defined a formal algebra around _Deltas_ (the encoded difference between any two states of a system) in a relational database. Their definition is actually generic enough to apply to any system whose state can be represented as a set of tuples, which is why it is still used in contemporary research on database versioning, including those involving non-relational data models (Khurana et al. [Storing and Analyzing Historical Graph Data at Scale](https://www.researchgate.net/publication/282403421_Storing_and_Analyzing_Historical_Graph_Data_at_Scale)). In order to support my deep dives into a couple of modern designs that use _Delta Arithmetic_ for versioning graph databases, I will spend some time in this post laying down its foundations, clarifying a few under-explained points along the way.

## Delta Arithmetic - The Ground Rules
We start with a database containing the following relations or tables, for the fictional inventory management system of a bicycle manufacturer:
1. `Suppliers` - A list of vendors and the parts they sell,
2. `Orders` - A record of orders placed to vendors, the parts and their quantities.

The contents of the two tables described above are shown below:

<table class="table table-hover" id="suppliers">
    <caption>Table 1: Suppliers</caption>
    <thead>
        <tr>
            <th>Supplier</th>
            <th>Part</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>Trek</td>
            <td>frame</td>
        </tr>
        <tr>
            <td>Campy</td>
            <td>brakes</td>
        </tr>
        <tr>
            <td>Trek</td>
            <td>pedals</td>
        </tr>
    </tbody>
</table>

<table class="table table-hover" id="orders">
    <caption>Table 2: Orders</caption>
    <thead>
        <tr>
            <th>Part</th>
            <th>Quantity</th>
            <th>Supplier</th>
            <th>Expected</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>frame</td>
            <td>400</td>
            <td>Trek</td>
            <td>8/31/93</td>
        </tr>
        <tr>
            <td>brakes</td>
            <td>150</td>
            <td>Campy</td>
            <td>9/1/93</td>
        </tr>
    </tbody>
</table>

### Tuple Representation
Every record in the database is mapped to a classified tuple of the form \\(RelName(field\\_value_1, field\\_value_2, ...)\\). For example the first row in the `Suppliers` table is represented as \\(Suppliers(Trek, frame)\\) and the first row in the `Orders` table as \\(Orders(frame, 400, Trek, 8/31/93)\\). The order of values in the tuple is the same as the order of field definitions in the tables above. In this way, every row in every table in the database is mapped to a tuple. For this small database, we can write the entire initial state \\(S_a\\) as a set of tuples, as shown below:

![Screenshot 2021-04-24 115725.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619245661306/1oYFhQB5F.png)

It should be noted at this point that none of the tables above sport row identifiers or primary keys. However, since the algebraic definitions assume the pure relational model, every tuple is considered unique in itself, and cannot exist more than once anywhere in the database. This constraint is captured mathematically by the property of sets that require every element to be unique. Additionally, if one or more of the fields in the tuple constitute a key, then at most one tuple with a particular combination of values for those fields can exist at a time in the state set. For example, in the `Orders` table, if the fields `(Part, Quantity, Supplier)` fields constituted the key (a bad design in reality, but will suffice for this example), then every tuple in the set \\(S_a\\), apart from being unique in itself, must also be unique w.r.t to the sub-tuple formed by the above 3 fields.

**Aside:** The simplest way to remediate the uniqueness problem without enforcing uniqueness in tuples with semantic (business-relevant) fields is to add a field representing a _dumb_ primary key. For example, a _Order ID_ field in the orders table. This also permits repeating values for semantic sub-tuples in the `Orders` set.

### Signed Atom
This is an expression of the form \\(\pm\langle\text{RelName}\rangle\langle\text{Tuple}\rangle\\) and corresponds to an insertion or deletion operation, depending on the \\(+\\) or the \\(-\\) prefix respectively. For example, \\(+Suppliers(Shimano, brakes)\\). This is the smallest unit of modification that can happen to the overall state set.

**Aside:** An _update_ operation would require use of two atoms:
1. One for deletion of the old value, and
2. One for insertion of the new value.

As far as the set algebra used for delta arithmetic is concerned, the order of the above two operations does not matter, as long as the _consistency constraints_ defined in the next section are met. Most databases, of course, allow atomic updates.

Since we're using sets to represent a pure relational model, **a signed atom representing insertion of a tuple already present in the current state results in a `No-Op`**. Similarly, **a signed atom representing deletion of a non-existent tuple from a set also results in a `No-Op`**.

### Delta
A delta is an **unordered**, finite set of signed atoms. For example,

![Screenshot 2021-04-24 115839.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619245747143/l41fBqSot.png)

#### Consistent and Failed Deltas
A delta is called _consistent_ if it does not contain both positive and negative versions of the same atom. Otherwise, it is called an inconsistent, or _failed_ delta. For example, the delta defined in (2) is a **consistent** delta, whereas a **failed** delta would look like:

![Screenshot 2021-04-24 115943.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619245797490/hculKE-mX.png)

Also, the delta being a set subject to the same uniqueness constraints imposed by keys (if present), we have the following: For a relation with two fields denoted by \\(R\[A, B\]\\), if \\(A\\) is the key and there exist two signed atoms \\(+R\[a, b\]\\) and \\(+R\[a, c\]\\) in a delta, then we must necessarily have \\(b = c\\) for the delta to be consistent (essentially collapsing them to a single signed atom).

**Aside:** One question that rose to my mind when I looked at (3) is why this should be disallowed. After all, the signed atoms within the delta are inverse operations of each other (insertion and deletion), and hence should just cancel each other out, resulting in a `No-Op` at worst. The paper does not directly address this question, though, as we shall see in a [later section](#whyinversesignedatomsleadtoafaileddelta), this restriction is  necessary to allow for delta operations to be safely applicable in any order (remember, the delta is an **unordered** set).

#### Delta Breakdown - Snapshot Fragments
The paper defines the following relations for a **consistent** delta \\(\Delta\\):

![Screenshot 2021-04-24 120359.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246055266/16TykLHDZ.png)

The consistency requirement can now be expressed as:

![Screenshot 2021-04-24 120439.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246094134/j7ZSYve-5.png)

For example, \\(\Delta_1\\) from (2) would be split into the following:

![Screenshot 2021-04-24 120538.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246157177/hfaFUz-5s.png)

Although the paper doesn't explicitly name the definitions in (4), I will label them here as **snapshot fragments**. These represent the same _type_ of elements in a set as the snapshot (tuples categorized by relation). The original delta, which represents events or operations, cannot be a direct algebraic operand along with the snapshot but its derivative snapshot fragments **can**.

Now that we have defined delta components that can directly combine with the snapshot algebraically, we define the _application_ of a delta \\(\Delta\\) to a snapshot \\(S\\) as the following equivalent functions:

![Screenshot 2021-04-24 120626.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246201075/3atMuMCBq.png)

#### Why Inverse Signed Atoms Lead to a Failed Delta
A few sections earlier, we saw that a delta of the form illustrated in (3) is a _failed_ delta, but did not elaborate on why it is so. Now that we have the commutative criterion of the delta function as described in (6), we can get a clearer picture.

We will examine two examples of failed deltas:
1. One where the signed atoms represent an element already present in current state, and
2. One where they represent a new element.

Say our current state has the following elements:

![Screenshot 2021-04-24 120754.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246299788/NK4FRdbF8.png)

First let us consider case 1. Let \\(\Delta_{f} = \left\\{
        \begin{array}{l}
            +Suppliers(Trek, frame), \\\\
            -Suppliers(Trek, frame)
        \end{array}
    \right\\}\\).
    
Applying (6), we get:

![Screenshot 2021-04-24 120855.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246351566/eop4DQuwS.png)

We also note that 
![Screenshot 2021-04-24 121048.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246465346/CBe0g6e5T.png)
 violating (5). To see if this delta still satisfies the commutative criterion of (6), we need:

![Screenshot 2021-04-24 121145.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246518295/zhRln38-b.png)

**which is a contradiction**.

Now let us consider case 2. Let \\(\Delta_{f'} = \left\\{
        \begin{array}{l}
            +Suppliers(Shimano, brakes), \\\\
            -Suppliers(Shimano, brakes)
        \end{array}
    \right\\}\\).
    
Applying (6), we get:

![Screenshot 2021-04-24 121524.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246737756/PKCiX29Ao.png)


We also note that 
![Screenshot 2021-04-24 121654.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246894616/SvTB3TYno.png)
violating (5). To see if this delta still satisfies the commutative criterion of (6), we need:

![Screenshot 2021-04-24 121833.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619246928031/Fuux-kOQi.png)

**which is also a contradiction**.

Therefore, we see that in order to satisfy the requirements in (6), we need to satisfy (5).

#### Delta Composition
Finally, I examine how the paper has formalized delta composition (or chaining) operations, thereby allowing us to apply at succession of deltas on a given state to arrive at the final state.

##### Smash Composition
One type of composition operation described is called a _smash_, denoted by \\("!"\\). This is the composition used by most active databases. Algebraically, a smash of two deltas is their union, with conflicts resolved in favour of the second argument. Given:

![Screenshot 2021-04-24 122123.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619247116262/PuQt0-iy2.png)

Then using (2) and (7) we get:

![Screenshot 2021-04-24 122254.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619247192064/Prjk5wo-3.png)

The formal definition for the _smash_ operation is given by:

![Screenshot 2021-04-24 122336.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619247237176/kNjm5j9U3.png)

The reader is encouraged verify whether the above holds true for our example, using (2), (7), (4), and plugging into (9).

The most important characteristic of the _smash_ composition is that it supports function composition, i.e.:

![Screenshot 2021-04-24 122424.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619247280310/1ZBBHW4sr.png)

##### Merge Composition
The second type of composition defined by the paper is the _merge_. This is less common in real world databases, and hence is not examined in detail. The _merge_ is denoted by \\("\\&"\\) and its formal definition is given by:

![Screenshot 2021-04-24 122501.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1619247315294/1WXVbJWs7.png)

Refer the paper for a slightly more detailed explanation of this.

## Summary
The authors have used elementary set algebra to come up with some elegant mathematical formalizations for representing database changes over time. Though they have presumed the presence of a pure relational context, there is nothing exceptional being done at the set algebra level - all its rules are strictly followed. This opens up the possibility of using delta arithmetic to analyze any kind of database whose states can be represented as sets of tuples. As we shall see in a future post, this is exactly what Khurana et al.[^2] have done when designing their _Temporal Graph Index_.