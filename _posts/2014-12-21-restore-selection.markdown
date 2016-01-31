---
layout: post
title:  "Restore the selection of a UITableViewCell after cancelling the Swipe to delete operation"
date:   2014-12-21 00:00:00 +0000
---

Yes, I’m back! In these months I’ve been focused on learning iOS 8, the Swift language and a lot of other things. So, forgive me in advance. In this post I will describe a simple trick to restore the selection of the current selected cell of a table view if the user cancels the <i>Swipe to delete</i> operation. Since I started to learn the new Apple language, code snippets will be written in Swift. Yeah!<br>Enabling the <i>Swipe to delete</i> functionality is quite simple. First of all, you need to set up a well functioning table view. For the sake of simplicity, I started from the <i>Single View Application</i> template where a <code>UITableView</code> has been set as a subview of the controller’s view shipped with that template. Obviously, the <code>ViewController</code> class has been set as the data source and the delegate for the table view. The AutoLayout constraints, the outlet connection and the data source/delegate have been configured by using Interface Builder and Storyboards. Here how the <code>ViewController</code> class looks like.

<pre>
import UIKit

class ViewController: UIViewController, UITableViewDataSource, UITableViewDelegate {

    @IBOutlet var tableView: UITableView!

    override func viewDidLoad() {
        super.viewDidLoad()

        self.tableView.registerClass(UITableViewCell.self, forCellReuseIdentifier: "MyCellIdentifier")
    }

    func tableView(tableView: UITableView, numberOfRowsInSection section: Int) -> Int {

        return 10
    }

    func tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCell {

        var cell: UITableViewCell = tableView.dequeueReusableCellWithIdentifier("MyCellIdentifier") as UITableViewCell
        cell.textLabel?.text = "Swipe to delete row"
        return cell
    }

    func tableView(tableView: UITableView, didSelectRowAtIndexPath indexPath: NSIndexPath) {

    }
}
</pre>

In order to enable the <i>Swipe to delete</i> functionality it is necessary to implement the following three methods:
<ul><li><code>- tableView:canEditRowAtIndexPath:</code></li>
<li><code>- tableView:commitEditingStyle:forRowAtIndexPath:</code></li>
<li><code>- tableView:editingStyleForRowAtIndexPath:</code></li>
</ul>While the first two belong to<code>UITableViewDataSource</code> and they are implemented respectively to ask if a row is editable and to commit the insertion or the deletion of a given row, the third is declared in the <code>UITableViewDelegate</code> and it is adopted to ask the editing style to use.

<pre>
func tableView(tableView: UITableView, canEditRowAtIndexPath indexPath: NSIndexPath) -> Bool {

    return true
}

func tableView(tableView: UITableView, commitEditingStyle editingStyle: UITableViewCellEditingStyle, forRowAtIndexPath indexPath: NSIndexPath) {

    if(editingStyle == UITableViewCellEditingStyle.Delete) {

        // Here you should commit your editing
    }
}

func tableView(tableView: UITableView, editingStyleForRowAtIndexPath indexPath: NSIndexPath) -> UITableViewCellEditingStyle {

    return UITableViewCellEditingStyle.Delete
}
</pre>

Quite easy, no? Ok, so now try to apply a swipe gesture (from right to left) to a row. A red button will appear in the right edge. Then, if you don’t decide to tap the delete button, the delete operation will be cancelled. <br>But what if a row was selected? In this case, cancelling the operation will make lose the selection. And I think it’s not what you want. Think, for example, of an iPad (or a iPhone+) application where the screen is divided into two parts (using a <code>UISplitVewController</code>). The left part presents a list by means of a table view while the right one shows the detail of the current selected row. If you have enabled the <i>Swipe to delete</i> functionality, you should want to maintain the state of the selection significant with the detail presented within the right part.
In order to achieve this scenario you need to rely on two additional methods that can be found in the <code>UITableViewDelegate</code> class:
<ul><li><code>- tableView:willBeginEditingRowAtIndexPath:</code></li>
<li><code>- tableView:didEndEditingRowAtIndexPath</code></li>
</ul>As stated by the documentation, they are respectively called when the user swipes horizontally across a row and when the table view exits editing mode after having been put into the mode by the user swiping across the row identified by <code>indexPath</code>. In the following, you can find how they are implemented.

<pre>
func tableView(tableView: UITableView, willBeginEditingRowAtIndexPath indexPath: NSIndexPath) {

    self.swipeGestureStarted = true;
}

func tableView(tableView: UITableView, didEndEditingRowAtIndexPath indexPath: NSIndexPath) {

    if(self.swipeGestureStarted) {
        self.swipeGestureStarted = false

        self.tableView.selectRowAtIndexPath(self.selectedIndexPath, animated: true, scrollPosition: .None)
    }
}
</pre>

As you can notice, these methods use two stored properties, <code>selectedIndexPath</code> and <code>swipeGestureStarted</code>, that are declared just below the <code>tableView</code> one.

<pre>
import UIKit

class ViewController: UIViewController, UITableViewDataSource, UITableViewDelegate {

    @IBOutlet var tableView: UITableView!

    private var swipeGestureStarted: Bool = false
    private var selectedIndexPath: NSIndexPath?

    // ...
}
</pre>

The <code>selectedIndexPath</code> property it is assigned to a new value each time the <code>- tableView:didSelectRowAtIndexPath:</code> is called.

<pre>
func tableView(tableView: UITableView, didSelectRowAtIndexPath indexPath: NSIndexPath) {

    self.selectedIndexPath = indexPath
}
</pre>

Instead, the <code>swipeGestureStarted</code> property allows to track whenever a swipe action starts and ends. In the start phase, the property is set to <code>true</code>.

<pre>
func tableView(tableView: UITableView, willBeginEditingRowAtIndexPath indexPath: NSIndexPath) {

    self.swipeGestureStarted = true
}
</pre>

In the end phase, the property, if <code>true</code> (we check if we previously applied a swipe gesture), is set to <code>false</code> and the selection (of the current selected row) is restored.

<pre>
func tableView(tableView: UITableView, didEndEditingRowAtIndexPath indexPath: NSIndexPath) {
    if(self.swipeGestureStarted) {
        self.swipeGestureStarted = false

        self.tableView.selectRowAtIndexPath(self.selectedIndexPath, animated: true, scrollPosition: .None)
    }
}
</pre>

As a you can notice, if the condition holds, i.e. the <code>swipeGestureStarted</code> is <code>true</code>, the property is set again to <code>false</code>. The assignment could be skipped but I'm experimenting a different behavior between iOS 7.1 and iOS 8.1. In particular, in iOS 8.1 the <code>- tableView:didEndEditingRowAtIndexPath:</code> is called twice. I'm not sure it is a bug or not but I already opened a radar.
